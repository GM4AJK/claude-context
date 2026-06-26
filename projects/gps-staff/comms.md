# GPS Staff — Communications Architecture

> **Current state:** Read `sdd/features/BASE.md`, `sdd/features/ROVER.md`, or `sdd/features/HANDHELD.md` in the gps-staff repo for the live capability index per unit. Individual feature specs in `sdd/features/` only need reading when actively working on that feature.

## System Diagram

```
Base F765 ──UART──► Base ESP32 Zero ──ESP-NOW──► Rover ESP32 Zero ──UART──► Rover F765
                         │                               │
                    SX1262 GFSK                     SX1262 GFSK
                    (RTCM TX)                       (RTCM RX)
                                                         │
                                                    BLE NimBLE
                                                         │
                                                   Handheld ESP32-S3
```

---

## Links

### GFSK/LoRa (SX1262) — Base → Rover RTCM

- **Purpose:** One-way RTCM correction stream from base to rover
- **Mode:** GFSK active bench mode (~7–8× less airtime than LoRa for 100m LOS)
- **Status:** Interrupt-driven DIO1 TX/RX working; `SetDIO3AsTCXOCtrl` + `CalibrateImage` still needed for Core1262-LF module
- **Framing:** `RTCM3_BUF_COUNT=4` — bump to 8 before adding PCB display driver (display flush ~33ms eats half overflow budget)
- **Test data:** `firmware/data/CambridgeSensoriisSample.bin` (39 frames, GPS+GLONASS MSM4)
- This link is one-way — RTCM flows base→rover only; commands go via ESP-NOW

### BLE NimBLE — Rover ESP32 ↔ Handheld

- **Purpose:** Bidirectional user comms — position stream + config/mode commands
- **Stack:** NimBLE (ESP-IDF component), bench-verified PR #187
- **Architecture:** Rover Zero = GATT peripheral/server; handheld = GATT central/client
- **Two characteristics on rover:**
  1. Position/status — NOTIFY only; rover publishes, handheld subscribes (pub/sub)
  2. Command — WRITE (handheld sends) + NOTIFY (rover acks/responds)
- **Service UUID:** 0xAB00, characteristic UUID: 0xAB01
- MTU negotiated to 512 bytes; notifications up to 500 bytes per chunk

**Key NimBLE IDF 6.x gotchas:**
- `htobs()` not available — on ESP32 (little-endian) use `0x0001` directly for CCCD subscribe
- `ble_gattc_disc_all_dscs(conn, chr_val_h, svc_end, cb, arg)` — NimBLE adds +1 to chr_val_h internally; do NOT pass chr_val_h+1
- Forward-declare all functions called before they are defined (warnings are errors)
- `esp_driver_uart` and `esp_driver_gpio` in CMakeLists REQUIRES — not `driver`

**sdkconfig.defaults keys for BLE:**
```
CONFIG_BT_ENABLED=y
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=512
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y   # Zero boards use USB-JTAG for console
```

### ESP-NOW — Base ESP32 Zero ↔ Rover ESP32 Zero

- **Purpose:** Bidirectional config/command relay between the two Zero boards
- **Why ESP-NOW:** No infrastructure needed, ~200m LOS, low latency, built into ESP32 radio
- **Status:** Not yet implemented (BLE bridge was the learning exercise; ESP-NOW is next)
- The STM32s on each board don't need to know about ESP-NOW — each ESP32 presents as a simple UART pipe to its F765

### Handheld → Base (via relay)

Handheld never connects to base directly. Path:
```
Handheld → BLE → Rover ESP32 → ESP-NOW → Base ESP32 → UART → Base F765
```
Acks travel the reverse path. This is intentional — one BLE link to manage, not two.

---

## ESP32-S3 Zero Firmware

**Single binary `firmware/esp32-base/`** handles both base and rover roles.

Role selection at boot:
- Read GP1 (PE2 on F765, pull-up); open = base, GND = rover
- PCB: F765 PE3 strobes GP2 to validate the reading
- Bench: GP1 hardwired link only; code times out after 500ms and reads GP1 directly

**Build & flash:**
```bash
cd firmware/esp32-base && /mnt/c/Users/kirkh/github/gps-staff/scripts/idf.sh build
/mnt/c/Users/kirkh/github/gps-staff/scripts/idf.sh -p /dev/esp32_base flash
```

Console via USB Serial/JTAG (`CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y`). UART0 (GPIO43/44) is the bridge to the Nucleo/F765.

---

## BLE as Pub/Sub (not serial)

BLE GATT is pub/sub by design — not a transparent bidirectional serial link:
- Rover advertises characteristics with NOTIFY property
- Handheld subscribes by writing 0x0001 to the CCCD ("subscribe me")
- Data flows rover→handheld as notifications; handheld→rover as GATT writes

For two-way traffic (commands), add a second characteristic with WRITE property. GATT Write + Notify on the same characteristic gives request/response semantics without a separate service.
