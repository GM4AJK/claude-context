# SOWB — Project Context

This document brings Claude Code up to speed on SOWB (Satellite Observer Workbench). Read this
before starting any work on this codebase.

**Project name:** SOWB
**GitHub repo:** https://github.com/GM4AJK/sowb

---

## Project Goal

A system with multiple inputs and outputs that adds OSD (on-screen display) information —
GPS-disciplined UTC timestamp, and satellite-mount pointing data — to two camera video feeds,
for observing satellites through a Skywatcher AltAz mount.

The project has two parallel tracks:

1. **Firmware** — STM32CubeIDE project (`firmware/`) for an STM32G491RE Nucleo, HAL-based
2. **Hardware** — KiCad PCB project (`hardware/sowb/`)

---

## Hardware Architecture

- **MCU:** STM32 Nucleo-G491RE
- **GPS:** GPS RX dev board — PPS signal on an EXT IRQ, NMEA sentences over UART for date/time
  and lock validity
- **OSD:** Two [MAX7456 OSD boards](https://github.com/sparkfun/On_Screen_Display_Breakout-MAX7456)
  (discontinued part, two BOB modules on hand), one per video channel, each on its own dedicated
  SPI bus + DMA (no bus sharing)
- **PC link:** USB Serial (PC comms) + G491 USB Serial (debug/console, via ST-Link VCP)
- **Skywatcher AltAz mount link:** PC talks to the mount via a YP-05 FT232RL USB-TTL breakout
  (VCCIO jumper set to 5V; originally planned around a CH340C, swapped for FT232RL hardware
  already on hand — same role). The G491 taps both directions of that link on two **RX-only**
  UARTs, purely to extract mount pointing data (alt/az) for the OSD — see Key Decisions below.

### Peripheral / pin map (STM32G491RETx, from `firmware/firmware.ioc`)

| Role | Peripheral | Notes |
|---|---|---|
| GPS PPS | GPIO EXTI, PB7 | Plain GPIO input in `.ioc` currently — EXTI trigger not yet configured |
| GPS NMEA | USART3 (PB11 RX, PB9 TX) | 9600 baud, FIFO enabled |
| Skywatcher tap, PC→mount | UART5 (PD2 RX) | 9600 baud, RX-only |
| Skywatcher tap, mount→PC | USART1 (PC5 RX) | 9600 baud |
| MAX7456 channel 1 | SPI3 + DMA1_Channel1 | CS PA15, VSYNC PC0, ODD/EVEN PC1, LOS PC2, HSYNC PC3, RST PA0 |
| MAX7456 channel 2 | SPI2 + DMA2_Channel1 | CS PB12, VSYNC PC6, ODD/EVEN PC7, LOS PC8, HSYNC PC9, RST PA8 |
| Debug/command console | USB VCP via ST-Link | — |

Both SPI buses: master, full-duplex, ~5.3 MBit/s, DMA memory→peripheral on TX only.

---

## Software Architecture (planned building blocks)

- **UTCClock** — struct + code holding a UTC date/time accurate to 1ms. Inputs: reset (init to
  2000-01-01T00:00:00.000), GPS PPS (EXT IRQ — resets ms counter, advances clock 1s), GPS NMEA
  (parsed for date/time + lock validity). API: `UTCCLOCK_GetDateTime()` returns a copy plus a
  GPS-valid flag.
- **MAX7456 driver** — abstract driver, instantiated twice (once per video channel). Text screen
  buffer; write char / C-string / length-prefixed string at x,y (wraps on overflow). On
  VSYNC + odd-frame IRQ: snapshot UTCClock, update the buffer, push to the MAX7456 via SPI DMA.
- **Debug/Command Monitor** — interactive console over the G491 USB serial port. Functionality
  TBD (e.g. write text into a MAX7456 buffer).

## Key Decisions

- **Superloop, not FreeRTOS.** No RTOS is configured. Time-critical work (GPS PPS, MAX7456
  VSYNC-triggered updates, Skywatcher byte capture) runs in ISRs/DMA callbacks; `main()`'s
  `while(1)` drives non-time-critical state machines off flags/ring buffers those ISRs set.
  Rationale: the only hard-real-time input (GPS PPS) wants deterministic ISR latency, not
  scheduler tick granularity; nothing in the system needs preemptive multitasking.
- **Skywatcher link is a passive RX-only tap, not an inline relay.** Two options were
  considered: relaying bytes through the G491 on two full serial ports (Option A), vs. wiring
  PC↔mount directly and tapping each direction into a G491 RX-only pin (Option B, chosen).
  Option A has an unavoidable ~1-byte-time latency floor at 9600 baud and — more importantly —
  would put the G491 inline in a live satellite-tracking control loop (PC sends variable-speed
  tracking updates to the mount), so any G491 firmware hiccup (from OSD/SPI/DMA or PPS work
  running concurrently) could degrade or stall mount tracking. The RX-only tap costs nothing
  extra and makes that structurally impossible.

---

## Development Workflow

Per `claude-context/workflows/WORKFLOW.md`, with SOWB-specific overrides:

- **No GitHub issues.** Features are speced in-repo (`specs/<slug>.md`), written through
  discussion and agreed *before* any code is written — not tracked as GitHub issues.
- **HAL-based**, not bare-metal register writes (differs from TwinTorqueESC/CLAUDEDEMO, which
  are bare-metal-only) — the CubeMX-generated code already uses HAL SPI/UART/DMA APIs.
- **User builds and flashes in STM32CubeIDE** — Claude does not build or run `make`, and does
  not flash the Nucleo (differs from CLAUDEDEMO, where Claude builds/flashes via PowerShell).
- Peripheral config lives in `firmware/firmware.ioc` (CubeMX) — do not hand-edit generated code
  outside `USER CODE BEGIN/END` markers, it's overwritten on next codegen.

---

## Current Status

- `firmware/` is CubeMX-generated scaffolding only (clock/peripheral init, empty `while(1)`) —
  no application code written yet for UTCClock, the MAX7456 driver, or the command monitor.
- `hardware/sowb/` is a KiCad PCB project (schematic/layout in progress).
- Repo initialized and pushed to GitHub (`GM4AJK/sowb`) with a root `CLAUDE.md` covering build
  commands, the peripheral/pin map, and this same development-workflow/decisions context.
- **Next step:** not yet defined — no design spec (`specs/`) opened yet.
