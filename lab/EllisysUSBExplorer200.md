# Ellisys USB Explorer 200 — Reference

**Device**: Ellisys USB Explorer 200 — non-intrusive USB 2.0 protocol analyzer
**Product page**: https://www.ellisys.com/products/usbex200/index.php
**Connection**: in-line between a USB host PC and the device under test — captures all bus traffic without affecting communication or device behaviour
**Software**: Ellisys USB Explorer desktop application (Windows) for live capture, decode, and analysis

---

## Capabilities

- Captures and decodes all three USB 2.0 speeds (Low/Full/High Speed) with automatic speed detection
- Decodes low-level bus states and protocol packets, plus high-level USB class decoders (e.g. Mass Storage)
- Identifies enumeration/protocol errors originating in the device, host controller, firmware, or drivers
- Measures bus performance/timing characteristics
- Two editions exist: **Standard** (essential capture/decode) and **Professional** (adds hardware triggering, extended class decoders, and a USB Analysis Development Kit for building custom analysis software/automation) — confirm which edition is on hand before assuming dev-kit/scripting features are available

---

## Why it's on the bench

Bought for verifying the gps-staff project's USB stack on the STM32F765 (see [[gps-staff]]):

- Enumeration verification (descriptors, configuration negotiation)
- CDC/MSC composite device debugging
- Validating the 100 mA → 500 mA current-draw transition during enumeration

---

## Notes

- Unlike the SDS824X/FY6800, this is **not** a SCPI/serial-scriptable instrument — it's primarily driven through its desktop GUI as a manual capture-and-analyse tool, so it doesn't fit the same "Python control class" pattern as the other lab docs. If the Professional Edition's development kit is available and gets used for programmatic capture, that would be the place to extend this doc with an automation reference.
