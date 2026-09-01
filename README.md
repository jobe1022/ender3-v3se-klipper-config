# Ender 3 V3 SE — Klipper Configuration

Working Klipper configuration for a Creality Ender 3 V3 SE, running on a
Raspberry Pi 5 host. This repo tracks `printer.cfg`, `mcu.cfg`,
`macros.cfg`, and supporting Moonraker/Mainsail/Crowsnest config.

For the firmware-level PRTouch bug fix (separate repo), see:
[ender3-v3-se-klipper-prtouch-fixed](https://github.com/jobe1022/ender3-v3-se-klipper-prtouch-fixed)

---

## Hardware

- **Host**: Raspberry Pi 5, booting from a Samsung 970 EVO NVMe SSD via
  M.2 HAT (not SD card). Required `PCIE_PROBE=1` in the EEPROM config to
  enable NVMe boot — this isn't on by default on early Pi 5 firmware.
- **Mainboard**: Creality stock silent board, `GD32F303RET6` MCU
  (STM32F103-compatible for Klipper flashing purposes). Confirmed via
  serial `M115`/`M503` queries — do not trust generic Ender 3 V3 SE specs
  online, this board's identity matters for picking the right Klipper
  `mcu` config.
- **Probe**: PRTouch — a load-cell-based Z probe built into this
  board/firmware combo, not a physical BLTouch pin sensor. It measures
  force via an HX711 amplifier, which matters later (see
  [Load Cell Sensitivity](#load-cell-sensitivity-important) below).
- **Firmware**: `jpcurti/ender3-v3-se-klipper-with-display`, built on
  `0xD34D/klipper_ender3_v3_se`. This fork adds PRTouch support *and*
  keeps the stock display working over a USART2 serial bridge — most
  other Klipper ports for this printer drop display support entirely.

## File Structure — Why Split

- **`printer.cfg`** — board-independent settings: kinematics, speed/accel
  limits, PID values, macros. This is the part that would carry over if
  the mainboard ever changed.
- **`mcu.cfg`** — board-specific pins and TMC2208 UART config, checked
  against the community reference at
  [0xD34D/ender3-v3-se-klipper-config](https://github.com/0xD34D/ender3-v3-se-klipper-config).
  Kept separate so a board swap only touches this file.
- **`macros.cfg`** — calibration and print-start/end macros, kept out of
  `printer.cfg` so it doesn't get lost in the sea of kinematic settings.

## Calibration Values (Current)

| Parameter | Value | How it was derived |
|---|---|---|
| `rotation_distance` (extruder) | 7.532 | Calculated from Marlin EEPROM `M92 E424.90`, not a spec default |
| BLTouch/PRTouch X/Y offset | -24.25 / -15.00 | From Marlin `M851` |
| Extruder PID (222°C) | Kp=27.046, Ki=1.541, Kd=118.664 | `PID_CALIBRATE` |
| Bed PID (60°C) | Kp=65.590, Ki=0.706, Kd=1522.518 | `PID_CALIBRATE` |
| Pressure advance (PLA) | 0.08 | Tuning tower measured 0.126 (FACTOR=0.005, caliper good zone 19.51–31.05mm), but dialed back to 0.08 after 0.126 correlated with visible gaps at curve/layer-change transitions |
| `square_corner_velocity` | 5.0 | Raised to 8.0 for corner-heavy geometry, then reverted to the original 5.0 alongside the pressure advance change — both were changed together after the gap defect appeared, so isolating which one actually caused it wasn't done |
| Z-offset, PLA (60°C bed) | 1.550 | PRTouch hot calibration |
| Z-offset, ABS (100°C bed) | 1.465 | PRTouch hot calibration — see note below |

**Why Z-offset differs between PLA and ABS:** the bed plate physically
expands as it heats. A `z_offset` measured at 60°C is *not* accurate at
100°C — the gap changed by ~0.03-0.09mm across various measurement
attempts, which is a meaningful fraction of a layer at 0.12mm resolution.
This is why there are separate hot-calibration macros per material
rather than one offset used for everything.

## Bed Mesh & Physical Shims
The bed was probed and found to have a genuine ~0.38mm tilt from front
to back / left to right (not a mesh artifact — mechanically confirmed,
mounting hardware was already tight). The V3 SE uses a rigid bolt-down
bed mount with no leveling springs (leveling is handled entirely via
probe + mesh compensation on this model), so shims were installe under the bed standoffs:

| Corner | Shim thickness |
|---|---|
| Front-left | 0.13mm |
| Front-right | none (reference) |
| Back-left | 0.38mm |
| Back-right | 0.34mm |

Bed mesh calibration was re-run after shim installation to capture the
corrected (much flatter) plane. Mesh compensation still handles the
residual few-hundredths-of-a-mm variance — the shims exist to remove the
*gross* tilt, not to achieve perfection on their own.

**Reinstall notes (rigid bolt-mount, no springs to self-equalize):**
loosen, shim, and retighten one bolt at a time rather than all four at
once, torque evenly in a cross pattern (front-left → back-right →
front-right → back-left), and avoid overtightening — with no spring to
absorb it, excess torque on a shimmed corner can flex the plate right at
that point and undo the correction.

## Macros

Four calibration macros, kept deliberately separate rather than
combined, since bed mesh and Z-offset are different jobs with different
sensitivity to timing:

- **`CALIBRATE_ABS`** — bed mesh at 100°C
- **`CALIBRATE_PLA`** — bed mesh at 60°C
- **`CALIBRATE_ZOFFSET_ABS`** — Z-offset probe at 100°C
- **`CALIBRATE_ZOFFSET_PLA`** — Z-offset probe at 60°C

Each macro:
1. Heats the bed to its target temp and waits for genuine stabilization
   (`TEMPERATURE_WAIT` + an extra dwell), not just a single momentary
   temp reading
2. Turns heaters off immediately before probing, to eliminate PWM
   switching noise on the load cell signal (see below)
3. Homes fresh
4. Runs the calibration
5. Reports the result but does **not** auto-`SAVE_CONFIG` — every new
   reading gets manually reviewed before being trusted, given the
   inconsistency documented below

## Load Cell Sensitivity (Important)

During ABS calibration, `PRTOUCH_PROBE_ZOFFSET` returned wildly
inconsistent results across consecutive runs in the same session —
readings of `1.517`, then `-2.340`, then `-2.530`, then finally a clean
`1.465` (validated via `PRTOUCH_ACCURACY`, std dev 0.015mm). Two root
causes were identified:

1. **ABS residue on the nozzle tip** — even a thin film changes the
   effective contact/trigger point for a force-based probe.
2. **Heater PWM noise** — the bed heater's rapid on/off switching to
   hold temperature can inject electrical noise into the load cell's
   HX711 amplifier signal if wiring runs near the bed's power leads.

**Practical takeaway:** always clean the nozzle before probing, and
always run `TURN_OFF_HEATERS` immediately before a PRTouch measurement
(both are baked into the calibration macros above). Always sanity-check
a new offset with `PRTOUCH_ACCURACY` before trusting it enough to
`SAVE_CONFIG` — a single bad reading with this probe type can be off by
several millimeters, not just a rounding error.

## PRTouch Firmware Fix

The upstream fork's `PRTOUCH_PROBE_ZOFFSET` command crashed on every run
due to a Python namedtuple immutability bug (`TypeError: 'probe_result'
object does not support item assignment`). This was fixed locally and is
tracked in the separate
[ender3-v3-se-klipper-prtouch-fixed](https://github.com/jobe1022/ender3-v3-se-klipper-prtouch-fixed)
repo, with a full writeup of the bug and fix. Moonraker's update manager
is set to `channel: dev` for the Klipper checkout on this Pi specifically
to prevent an auto-update from silently reverting this patch.

## Tags

- **`stock-head-profile`** — full working config as of the pre-overhaul
  baseline: stock toolhead, PID tuned, PA 0.08, ABS Z-offset 1.465, bed
  shims installed. Use `git checkout stock-head-profile -- printer.cfg
  mcu.cfg` to pull this exact state back for comparison after toolhead
  changes.
