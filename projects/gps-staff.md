# GPS Staff — Overview

**Repo:** `/mnt/c/Users/kirkh/github/gps-staff` | `https://github.com/GM4AJK/gps-staff`
**Living spec:** `sdd/README.md` (full technical detail — not the root README)

---

## What It Is

A DIY RTK GNSS survey staff — base + rover pair, identical custom PCBs. User plants the base, configures it from a handheld, walks with the rover to survey. The handheld (separate ESP32-S3 device) is the only UI.

## Boards at a Glance

| Board | Chip | Role | Firmware |
|-------|------|------|----------|
| PCB staff ×2 | STM32F765VIT6 + ZED-F9P + SX1262 | Base / rover (identical PCB) | `rtk-base/`, `rtk-rover/` — CubeIDE, user builds |
| Nucleo-F767ZI | STM32F767ZI | Bench base sim | `Nucleo-F767ZI-FreeRTOS/` — CubeIDE |
| Nucleo-F446RE | STM32F446RE | Bench rover sim | CubeIDE |
| ESP32-S3 Zero ×2 | ESP32-S3 | Staff comms bridge | `firmware/esp32-base/` — Claude builds via `scripts/idf.sh` |
| Handheld | ESP32-S3 4.3" touch | UI controller | `firmware/esp32-handheld/` — Claude builds |

## Two-Phase Operation

1. **Base setup:** Plant base → configure (known coords or F9P self-survey) → start streaming
2. **Survey:** Walk with rover + handheld. Base is a dumb RTCM transmitter. "Base: OK" = RTCM flowing.

## Current State — Covering Files

For live capability status per unit, read these files in the gps-staff repo (faster than scanning individual feature specs):

| File | Covers |
|------|--------|
| `sdd/features/BASE.md` | esp32-base (base role) + STM32 base firmware |
| `sdd/features/ROVER.md` | esp32-base (rover role) + STM32 rover firmware |
| `sdd/features/HANDHELD.md` | esp32-handheld |

Individual feature specs live in `sdd/features/nnnnn-name.md` — only read when actively working on that feature.

## Snapshot Status (2026-06-26)

| Area | Status |
|------|--------|
| PCB v1.0 | Ordered JLCPCB 2026-06-25, awaiting delivery |
| ZED-F9P | Ordered DigiKey 2026-06-25 |
| BLE bridge (Zero boards) | Bench verified PR #187 |
| GFSK RTCM OTA | Active bench mode, interrupt-driven |
| Handheld display + touch | Working, portrait 270°, LVGL 8.4 |
| Handheld UI | Early sandbox — battery indicator widget only |
| MMC5603NJ driver | Needed (replaced discontinued LIS3MDL) |
| Tilt fusion / soft-iron cal | Deferred |
| FatFS RTC timestamp | Deferred until real F9P arrives |
| F765 PE2/PE3 mode-select GPIO | Not yet implemented, waiting for PCB |

## GitHub Workflow

- SSH broken from WSL — HTTPS + `gh` PAT only. See repo `CLAUDE.md`.
- **PR-only ruleset** — branch → commit → `gh pr create` → `gh pr merge`. No direct pushes to `main`.
- Datasheets: `docs/datasheets/` — update `docs/README.md` catalog in same PR.
