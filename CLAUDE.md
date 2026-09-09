# Claude Context — Session Bootstrapper

You have just been asked to read this file. Do not read anything else yet.

Ask the user the following questions **one at a time**, reading the indicated file only after each answer narrows the scope. Stop as soon as you have enough context to be useful — do not read files that aren't relevant to today's work.

---

## Step 1 — What are we working on?

Ask: **"I've read claude-context/CLAUDE.md. What are we working on today?"**

| Answer | Read next |
|--------|-----------|
| GPS Staff / gps-staff | `projects/gps-staff.md` (overview), then go to Step 2 |
| TwinTorqueESC / ESC | `projects/TwinTorqueESC.md` — done, no Step 2 |
| DigitalScales / scales | `projects/DigitalScales.md` — done, no Step 2 |
| ClaudeDemo / demo | `projects/CLAUDEDEMO.md` — done, no Step 2 |
| SOWB / Satellite Observer Workbench | `projects/SOWB.md` — done, no Step 2 |
| Lab instrument / Python scripts | go to Step 1b |
| Git / GitHub / workflow | `workflows/WORKFLOW.md` — done, no Step 2 |

### Step 1b — Which instrument?

Ask: **"Which instrument?"**

| Answer | Read |
|--------|------|
| Oscilloscope / SDS824X | `lab/SDS824X.md` |
| Signal generator / FY6800 | `lab/FY6800.md` |
| USB analyser / Ellisys | `lab/EllisysUSBExplorer200.md` |

---

## Step 2 — GPS Staff only: which area?

After reading `projects/gps-staff.md`, ask: **"Which area of GPS Staff are we focusing on?"**

| Answer | Read next |
|--------|-----------|
| Handheld / LVGL / display / UI | `projects/gps-staff/handheld.md`, then `sdd/features/HANDHELD.md` in gps-staff repo |
| STM32 / firmware / Nucleo / F765 / F767 / F446 / base / rover | `projects/gps-staff/stm32-firmware.md`, then `sdd/features/BASE.md` or `ROVER.md` as relevant |
| Hardware / PCB / schematic / components / pins | `projects/gps-staff/hardware.md` |
| Comms / LoRa / GFSK / BLE / ESP-NOW / radio | `projects/gps-staff/comms.md` |
| ESP32 Zero / base bridge / rover bridge | `projects/gps-staff/comms.md`, then `sdd/features/BASE.md` in gps-staff repo |
| General / not sure | No further reading needed — overview is enough to start |

After reading a covering file (`BASE.md`, `ROVER.md`, `HANDHELD.md`), only read an individual feature spec from `sdd/features/` if that specific feature is what we are actively working on today.

---

## What not to do

- Do not read all files speculatively.
- Do not read sub-files before asking Step 2.
- Do not summarise this file back to the user — just ask the questions.
