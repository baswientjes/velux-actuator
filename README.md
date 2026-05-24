# velux-actuator

DIY actuator for Velux pivot roof windows — ESP32 + NEMA17 stepper + A4988 driver,
integrated with HomeAssistant via ESPHome.

---

## Features

### Hardware control (ESPHome firmware)
- **Stepper motor drive** via A4988 — STEP/DIR/EN interface, 1/16 microstepping
- **VM rail isolation** via IRF9540N P-channel MOSFET — protects A4988 from back-EMF
  when window is moved manually while unpowered; off by default at boot
- **Safe shutdown/boot sequence** — motor always disabled before VM is cut, and VM
  always enabled before motor is enabled, per A4988 datasheet requirements
- **Homing on boot** — drives to the closed endstop, zeroes the step counter, then
  resumes normal operation
- **Dual endstops** — NC microswitches at fully-closed and fully-open positions,
  debounced at 20 ms
- **Window angle sensor** — MPU-6050 accelerometer on the sash, gives true angle
  independent of step count via I2C

### HomeAssistant integration
- **Cover entity** — standard HA `cover` with open / close / stop / position support
- **Sleep mode** — blocks all automatic adjustments while someone is sleeping in the
  attic; configurable duration from 1–10 hours (default 10 h), auto-cancels when the
  timer expires or can be turned off manually at any time
- **Rain / storm close** — automatically closes when the weather integration reports
  rain, pouring, lightning, or hail
- **Heat venting** — automatically opens when the attic is too warm and outside air is
  cooler; only opens in fair weather conditions
- **Cold weather close** — automatically closes when outside temperature drops below a
  configurable threshold, to avoid venting warm air
- **High wind close** — automatically closes when strong wind is forecast
- All automations respect sleep mode — the window will not move automatically while
  sleep mode is active, regardless of weather conditions

### Manual override
Always possible — the motor and driver can be unpowered at any time without risk to
electronics; the IRF9540N gate is pull-up to 12V, keeping VM disconnected by default.

---

## Hardware

| Component | Part |
|---|---|
| Microcontroller | ESP32 DevKit (ESP-WROOM-32) |
| Stepper motor | NEMA17, ~50 Ncm holding torque |
| Stepper driver | Pololu A4988 |
| VM switch | IRF9540N P-channel MOSFET (high-side, 12V rail) |
| Angle sensor | MPU-6050 (I2C, mounted on sash) |
| Endstops | NC microswitches × 2 |
| Drive train | GT2 belt + rack and pinion |
| Power supply | 12V DC, ≥ 3A |

## GPIO assignments (ESP32 DevKit)

| GPIO | Function |
|---|---|
| GPIO26 | A4988 STEP |
| GPIO27 | A4988 DIR |
| GPIO14 | A4988 EN (active LOW) |
| GPIO13 | P-FET gate drive (LOW = VM enabled) |
| GPIO21 | I2C SDA (MPU-6050) |
| GPIO22 | I2C SCL (MPU-6050) |
| GPIO32 | Closed endstop (NC, internal pull-up) |
| GPIO33 | Open endstop (NC, internal pull-up) |

---

## Repository layout

```
firmware/
  velux-actuator.yaml   ESPHome firmware config
  secrets.yaml          WiFi / API / OTA credentials (not committed)
homeassistant/
  velux_helpers.yaml    HA helpers and automations (paste into config.yaml or packages)
```

---

## Setup

### Firmware

```bash
# Validate config (no device needed)
esphome config firmware/velux-actuator.yaml

# Flash over USB (first time)
esphome run firmware/velux-actuator.yaml

# OTA update (after first flash)
esphome run firmware/velux-actuator.yaml --device <ip-address>
```

### HomeAssistant

1. Copy the contents of `homeassistant/velux_helpers.yaml` into your `configuration.yaml`,
   or drop it as a package file under `packages:`.
2. Adjust the entity IDs at the top of the file if your ESPHome device name differs.
3. Replace `sensor.attic_temperature` with your actual indoor temperature sensor entity.
4. Adjust temperature thresholds and weather conditions to your preference.
5. Reload HA configuration and add the new helpers to your dashboard.

---

## Open items (resolve before finalising firmware)

1. Window force measurement — kitchen scale at lower sash edge (static + mid-travel)
2. Gear ratio — derive from measured force, motor torque, pinion radius, safety factor 3×
3. Pinion diameter — choose based on required torque and rack pitch
4. Total step count — measure after mechanical installation (`max_position` in firmware)
5. DIR polarity — confirm which DIR state = open vs close on first bench test
6. MPU-6050 axis — confirm which axis (pitch/roll) = window angle after mounting
7. NEMA17 exact model — confirm rated current for VREF calculation
8. Endstop GPIO assignment — GPIO32/33 assigned; confirm during commissioning
