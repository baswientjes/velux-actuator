# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Guidance for Claude

### Current project phase
This project has **no code yet**. Hardware design is largely settled; firmware has not been written. The immediate next steps are mechanical commissioning and resolving the open items listed at the bottom of this file.

### Scope rules
- Do not fill in TBD values (GPIO assignments, gear ratio, max_position, VREF) — ask the user to provide measured or confirmed values first.
- Do not generate final firmware until all open items are resolved. Drafts and scaffolding are fine, but mark TBD placeholders explicitly.
- Always respect the A4988 shutdown/boot sequence when writing any firmware or pseudocode involving motor or VM control.

### ESPHome development commands
Once a `.yaml` config file exists:
```bash
# Validate config (no device needed)
esphome config <file>.yaml

# Flash over USB (first time)
esphome run <file>.yaml

# OTA update (after first flash)
esphome run <file>.yaml --device <ip-address>
```

---

# Velux window actuator — project context

This file provides Claude Code with full context for the DIY Velux pivot roof window
actuator project. Read it completely before making any suggestions or writing any code.

---

## Project goal

Automate opening and closing of a Velux pivot roof window in an attic using a stepper
motor and rack-and-pinion drive train, with HomeAssistant integration via ESPHome.
Manual override must always be possible without risk to the electronics.

---

## Hardware

### Microcontroller
- **NodeMCU ESP8266** (ESP-12E form factor)
- Powered from 12V rail via AMS1117-3.3 LDO (or via USB during development)
- All GPIOs are 3.3V logic

### Stepper motor
- **NEMA17**, ~50Ncm (5kg·cm) holding torque
- Specific model TBD — to be confirmed before firmware tuning

### Stepper driver
- **Pololu A4988** module (chosen over TMC2209 for simplicity; noise/precision not a concern)
- STEP/DIR/EN interface only — no UART
- Microstepping: MS1/MS2/MS3 all HIGH → 1/16 step (default, may be revised)
- VREF set via onboard trimmer pot for target current (~1.5A RMS for NEMA17)
- VDD powered from 3.3V rail

### VM isolation switch (back-EMF protection)
- **IRF9540N** P-channel MOSFET, high-side switch on the 12V VM rail
- Gate pulled to 12V via 10kΩ (Rpu) — FET off by default when MCU is unpowered
- Gate driven via GPIO12 (D6) through 220Ω series resistor (Rg)
- GPIO pulls gate LOW to turn FET ON (enable VM to driver)
- Protects A4988 from back-EMF when window is closed manually without power
- **Bulk capacitor**: 2200µF / 25V electrolytic across VM/GND after FET drain
- **TVS diode**: SMBJ28CA bidirectional across VM/GND after FET drain

### Power supply
- 12V DC, ≥3A (lab PSU during development, switched-mode brick in production)

### Endstops (homing)
- Two **microswitches**, wired **normally-closed (NC)**
- One at fully-closed position, one at fully-open position
- Connected to ESP8266 GPIOs with internal pull-ups enabled
- Debounced in ESPHome via `delayed_on` filter (~20ms)
- On boot: home to closed endstop, zero step counter, then resume last known state

### Mechanical drivetrain
- **GT2 timing belt** from motor shaft (small pulley) to pinion shaft (large pulley)
- Pinion meshes with linear rack attached to lower edge of Velux sash
- **Gear ratio**: TBD — pending force measurement of window (kitchen scale test)
- Design safety factor: **3×** measured static force (accounts for wind load, weatherstrip
  friction, belt/pinion losses ~15-20%, and worst-case suction on partially open sash)
- Speed is not a consideration — reliability and torque are the priority
- Usage: very low duty cycle (a few times per day at most, typically once or twice a week)
  — thermal performance of A4988 is a complete non-issue

---

## GPIO pin assignments (NodeMCU ESP8266)

| Pin label | GPIO | Function |
|-----------|------|----------|
| D1 | GPIO5 | A4988 STEP |
| D2 | GPIO4 | A4988 DIR |
| D5 | GPIO14 | A4988 EN (active LOW) |
| D6 | GPIO12 | P-FET gate drive (LOW = VM enabled) |
| TBD | TBD | Closed endstop (NC microswitch) |
| TBD | TBD | Open endstop (NC microswitch) |

**Avoid boot-sensitive pins**: GPIO0, GPIO2, GPIO15 must not be driven by external
hardware during boot. Do not assign endstops or any other function to these pins.

**Remaining safe GPIOs for endstops**: GPIO13 (D7), GPIO16 (D0) — confirm before use.
GPIO16 has no internal pull-up; use external 10kΩ if assigned to an endstop.

---

## A4988 wiring notes

- MS1, MS2, MS3 → all HIGH (3.3V) for 1/16 microstepping
- EN pin: active LOW — pulled HIGH (motor disabled) via 10kΩ to 3.3V by default
  MCU asserts LOW to enable motor
- STEP: one rising edge = one microstep
- DIR: HIGH or LOW sets direction (confirm which is open vs close during commissioning)
- VDD: 3.3V logic supply
- VM: 12V motor supply (switched via P-FET)
- VREF: set trimmer so VREF ≈ 0.75V for ~1.5A (formula: I = VREF / (8 × Rs),
  where Rs = 0.068Ω on most Pololu A4988 modules — verify on your specific module)

### Critical A4988 shutdown sequence
Per A4988 datasheet: never switch VM while motor coils are energised.
Firmware **must** follow this order:
1. Assert EN HIGH (disable motor outputs) — wait ≥1ms
2. Then cut VM (pull GPIO12 HIGH to turn off P-FET)

Boot sequence is the reverse:
1. Enable VM (GPIO12 LOW)
2. Wait ≥1ms
3. Assert EN LOW (enable motor)

---

## Firmware

### Platform
- **ESPHome** — native HomeAssistant integration, OTA updates, stepper component built-in
- Target board: `esp8266` / `nodemcu`

### HomeAssistant integration
- Window exposed as a **`cover`** entity (supports open/close/stop/position)
- Automation triggers: rain sensor input and/or weather API data in HomeAssistant
- Rain/wind response: close window automatically when triggered
- Position tracking: step-count based, zeroed on homing

### ESPHome components to use
- `stepper`: `a4988` driver, STEP/DIR pins, acceleration/deceleration ramp
- `binary_sensor`: both endstops, `device_class: window` or generic, NC wiring
- `cover`: custom cover using stepper position + endstop state
- `output` or `switch`: GPIO12 for P-FET VM control, linked to motor enable logic
- `on_boot`: trigger homing sequence

### Homing sequence (on boot)
1. Enable VM (GPIO12 LOW)
2. Enable motor (EN LOW)
3. Drive slowly toward closed endstop
4. When closed endstop triggers: zero step counter, stop motor
5. Motor is now in known position — ready for HomeAssistant commands

### Key parameters (to be filled in after mechanical commissioning)
- `max_position`: total steps from closed to fully open (measure after install)
- `acceleration`: start conservative (~50 steps/s²), tune on bench
- `deceleration`: same or slightly higher than acceleration
- Endstop debounce: 20ms `delayed_on`

---

## Open items (resolve before finalising firmware)

1. **Window force measurement**: use kitchen scale to measure force at lower sash edge,
   both static (break-away from closed) and mid-travel. Multiply by 3 for design load.
2. **Gear ratio**: calculate from measured force, motor torque (50Ncm), pinion radius,
   and safety factor. Determine GT2 pulley tooth counts accordingly.
3. **Pinion diameter**: choose based on required torque and available rack pitch.
4. **Endstop GPIO assignment**: assign from remaining safe pins, note GPIO16 caveat.
5. **NEMA17 exact model**: confirm rated current for VREF calculation.
6. **Total step count**: measure after mechanical installation for `max_position`.
7. **DIR polarity**: confirm which DIR state = open vs close during first bench test.

---

## What this project is NOT

- Not a precision/silent application — A4988 noise is irrelevant
- Not time-critical — slow movement is fine and preferred
- Not high duty cycle — thermal design is a non-issue
- Not battery powered — always mains via 12V PSU