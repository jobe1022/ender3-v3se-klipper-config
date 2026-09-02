# Ender 3 V3 SE — Klipper Configuration

Working Klipper configuration for a Creality Ender 3 V3 SE, running on a
Raspberry Pi 5 host, currently mid-overhaul with a ceramic hotend, linear
rails, USB accelerometer, and several other mods on top of the stock
PRTouch-enabled board.

For the firmware-level PRTouch bug fix (separate repo), see:
[ender3-v3-se-klipper-prtouch-fixed](https://github.com/jobe1022/ender3-v3-se-klipper-prtouch-fixed)

---

## Hardware

- **Host**: Raspberry Pi 5, booting from a Samsung 970 EVO NVMe SSD via
  M.2 HAT (not SD card).
- **Mainboard**: Creality stock silent board, `GD32F303RET6` MCU
  (STM32F103-compatible for Klipper flashing purposes).
- **Probe**: PRTouch — a load-cell-based Z probe (HX711 amplifier), not a
  physical BLTouch pin sensor. See [Load Cell Sensitivity](#load-cell-sensitivity)
  below — this probe type is meaningfully more sensitive to heat and
  electrical noise than a standard inductive/optical probe.
- **Firmware**: `jpcurti/ender3-v3-se-klipper-with-display`, built on
  `0xD34D/klipper_ender3_v3_se`.
- **Hotend**: Official Creality Ender 3 V3 SE/KE ceramic heating block
  upgrade kit (quick-swap nozzle, rated 300°C, up to 600mm/s-class flow).
  Replaced the stock hotend as part of a full toolhead overhaul.
- **X/Y motion**: Converted from stock wheel carriages to linear rails.
- **Accelerometer**: BigTreeTech USB ADXL345 v2.0 (RP2040-based), mounted
  on the extruder stepper (moves rigidly with the toolhead carriage — a
  valid, if not perfectly ideal, mounting position for input shaping on a
  direct-drive setup).
- **Filament sensor**: mechanical runout switch wired to the board's
  dedicated `FILAM` header (silkscreen-labeled, next to the DC24V/fuse
  area) — see [Filament Sensor Pin](#filament-sensor-pin) below.
- **Bed leveling**: rigid bolt-down mount, no springs (stock for this
  model). Physical shims added under the bed standoffs to correct a
  measured ~0.38mm tilt — see [Bed Shims](#bed-shims-and-mesh) below.

## File Structure — Why Split

- **`printer.cfg`** — board-independent settings: kinematics, speed/accel
  limits, PID values, macros.
- **`mcu.cfg`** — board-specific pins and TMC2208 UART config.
- **`macros.cfg`** — calibration and filament-sensor macros.

## Calibration Values (Current)

| Parameter | Value | Notes |
|---|---|---|
| `rotation_distance` (extruder) | 7.532 | From Marlin EEPROM `M92 E424.90` |
| Extruder PID (245°C, ceramic hotend) | Kp=22.936, Ki=2.731, Kd=48.167 | Re-tuned after hotend swap — meaningfully different thermal mass than stock, so a different PID response is expected, not a bug |
| Bed PID (60°C) | Kp=65.590, Ki=0.706, Kd=1522.518 | Unchanged — bed itself wasn't modified |
| `max_temp` (extruder) | 300 | Raised from stock 260 to use the ceramic hotend's actual rating |
| Pressure advance (PLA, 195°C) | **0.09** | Measured via `TUNING_TOWER`, official Klipper/Creality method (`SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=1 ACCEL=500` first, then `TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.005`). Superseded the old stock-hotend value of 0.08 — different hotend melt-zone geometry genuinely changes PA, don't assume the old value transfers. |
| Pressure advance (ABS) | *not yet re-tuned* | Old stock-hotend value was 0.08; needs its own tuning tower run on the ceramic hotend before trusting it |
| Z-offset, PLA (60°C bed) | 1.930 | Re-measured after ceramic hotend + bed shims — a ~0.4mm jump from the old 1.550 is *expected* here, not an error, since both hotend geometry and bed plane changed |
| Z-offset, ABS (100°C bed) | 1.465 | Pre-hotend-swap value — **needs re-measuring** with `CALIBRATE_ZOFFSET_ABS` before trusting it on the new hotend |
| `max_velocity` | 350 | Ceiling, not a target — real per-feature print speeds stay well under this |
| `max_accel` | 5400 | Set at the Y-axis input-shaping limit (see below), not higher — going past this increases shaper "smoothing" and can reintroduce ringing |
| `square_corner_velocity` | 8.0 | Restored to 8.0 after being reverted to 5.0 earlier — the original defect that caused the revert was later traced to a Z-offset/thermal-expansion problem, not this setting |
| `idle_timeout` | 18000 (5 hours) | Increased from Klipper's 600s default after a paused print (filament runout mid-print) hit the timeout, disabled steppers, and lost position tracking — see [Lessons](#lessons--gotchas) below |

### Input Shaping

Measured via `TEST_RESONANCES` + `calibrate_shaper.py` (Klipper's official
resonance testing tools — required manually installing `numpy` and
`matplotlib` into `~/klippy-env` first, since they aren't included by
default):

```
X: shaper_type_x = 2hump_ei, shaper_freq_x = 83.2 Hz  (0% vibration, max_accel <= 7300)
Y: shaper_type_y = mzv,      shaper_freq_y = 43.8 Hz  (0% vibration, max_accel <= 5400)
```

Both axes came back with clean, single-dominant-frequency resonance
signatures — a good indirect sign that mounting the accelerometer on the
extruder stepper (rigid with the toolhead) was giving a trustworthy
signal, not noisy garbage.

## Bed Shims and Mesh

The bed was probed and found to have a genuine ~0.38mm tilt front-to-back
and left-to-right (not a mesh artifact — mechanically confirmed, all bed
standoffs were already torqued). The V3 SE uses a rigid bolt-down bed
mount with no leveling springs (leveling is handled entirely via probe +
mesh compensation on this model), so custom-thickness shims were
designed and printed to sit under the bed standoffs:

| Corner | Shim thickness |
|---|---|
| Front-left | 0.13mm |
| Front-right | none (reference — highest point) |
| Back-left | 0.38mm |
| Back-right | 0.34mm |

**Reinstall notes (rigid mount, no springs to self-equalize):** loosen,
shim, and retighten one standoff at a time rather than all four at once;
torque evenly in a cross pattern (front-left → back-right → front-right
→ back-left); avoid overtightening, since with no spring to absorb it,
excess torque on a shimmed corner can flex the plate right at that point
and undo the correction.

**Result:** corner-to-corner spread went from **0.38mm → 0.14mm** (~63%
reduction) after shimming — real, measurable improvement, though not
perfectly flat. The residual variance is small enough that bed mesh
compensation handles it cleanly during printing.

Both the mesh *and* Z-offset are calibrated **hot** (at actual print
temperature, heater turned off immediately before probing) — see
[Load Cell Sensitivity](#load-cell-sensitivity) for why this matters more
than it would with a standard probe.

## Calibration Macros

Four macros, kept deliberately separate since bed mesh and Z-offset are
different jobs with different sensitivity to timing:

- **`CALIBRATE_MESH_PLA`** / **`CALIBRATE_MESH_ABS`** — bed mesh at
  60°C / 100°C, auto-loads the result as the active profile
  (`BED_MESH_PROFILE LOAD=default`) so it doesn't just sit saved-but-unused
- **`CALIBRATE_ZOFFSET_PLA`** / **`CALIBRATE_ZOFFSET_ABS`** — Z-offset
  probe at 60°C / 100°C, includes a built-in `PRTOUCH_ACCURACY`
  consistency check before reporting the result

Each macro: heats the bed and waits for genuine stabilization
(`TEMPERATURE_WAIT` + an extra ~3 minute dwell for full thermal
expansion), turns heaters off immediately before probing (kills PWM
switching noise on the load cell), homes fresh, then runs the actual
calibration. None of them auto-`SAVE_CONFIG` — every new reading gets
manually reviewed against its accuracy check before being trusted.

## Filament Sensor Pin

The board has a dedicated, silkscreen-labeled `FILAM` header (separate
from the Z-limit endstop port) — but its actual GD32 pin is **not
documented anywhere publicly** as of this writing: not in the community
`0xD34D/ender3-v3-se-klipper-config` reference, not in any of several
"complete" community config forks checked, and it's an open, unanswered
question on Creality's own owners forum (asked January 2026, still
unanswered as of this repo's last update).

**It was found empirically**, by declaring a batch of `[gcode_button]`
test entries on every plausible unused GD32F303 GPIO pin at once,
restarting, then physically triggering the sensor and watching for a
`TRIGGERED`/`RELEASED` response tied to the physical action (not the
random noise every floating pin shows briefly at boot).

**Result: the `FILAM` header's signal pin is `PC15`.**

```
[filament_switch_sensor filament_sensor]
switch_pin: !PC15
pause_on_runout: True
runout_gcode:
    M117 Filament runout detected!
    PAUSE
insert_gcode:
    M117 Filament inserted.
```

## USB Accelerometer Setup

Two separate, non-obvious problems had to be solved to get the BTT USB
ADXL345 working on this fork — worth documenting in full since both are
easy to hit and neither is specific to this exact printer.

### 1. MCU protocol version mismatch

Symptom: `Klipper reports: SHUTDOWN — MCU Protocol error ... Command
format mismatch: query_adxl345 oid=%c rest_ticks=%u vs query_adxl345
oid=%c clock=%u rest_ticks=%u`

Cause: the accelerometer's RP2040 firmware, if flashed using generic/
mainline Klipper build instructions (as most tutorials assume), reports a
mainline version string (`v0.11.0-...`). This fork's host firmware
reports its own custom version string (`1.0.1-...`). **Two different
Klipper codebases, incompatible wire protocol** — even though both sides
are "working" Klipper firmware individually.

Fix: rebuild the accelerometer's firmware from the *same* `~/klipper`
source tree the mainboard uses, targeting RP2040 instead of the board's
STM32F103-compatible architecture:

```bash
cd ~/klipper
cp .config .config.mainboard-backup   # save the mainboard's build config first!
make menuconfig                       # set Micro-controller Architecture -> Raspberry Pi RP2040
make clean && make                    # produces out/klipper.uf2

# Put the accelerometer in bootloader mode (hold BOOTSEL while plugging in USB),
# then flash it:
sudo mount /dev/sda1 /mnt/rp2040      # adjust device as needed; look for INFO_UF2.TXT
sudo cp ~/klipper/out/klipper.uf2 /mnt/rp2040/

# IMPORTANT: restore the mainboard's build config afterward, or the next
# `make` for the mainboard will build the wrong architecture:
cp .config.mainboard-backup .config
make clean && make
```

Note: reflashing from this source tree regenerates the device's USB
identity string, so `/dev/serial/by-id/` will show a new
`usb-Klipper_rp2040_<serial>-if00` path — update `[mcu accelerometer]`'s
`serial:` line in `printer.cfg` accordingly.

### 2. Wrong pin/SPI config for this specific board

Once the protocol issue was fixed, the accelerometer still failed with
`Invalid adxl345 id (got ff vs e5)` — a chip mismatch error, but the chip
was confirmed correct (ADXL345 V2.0, per board markings). The actual
issue was a wrong `cs_pin`/`spi_bus` guess. **BTT's actual documented
config for this board:**

```
[adxl345]
cs_pin: accelerometer:gpio9
spi_bus: spi1_gpio8_gpio11_gpio10
axes_map: -x,-y,-z
```

### 3. USB bus sharing caused intermittent MCU disconnects

Symptom: random `Failed automated reset of MCU 'mcu'` / `Lost
communication with MCU 'mcu'` shutdowns, especially under load, with no
obvious cause in the logs beyond a `next_clock` counter reset (a genuine
MCU-level reset, not just a comms hiccup).

Cause: `lsusb -t` revealed the mainboard and the accelerometer were
sharing the same physical USB 2.0 controller (`Bus 003`, a 2-port
controller on the Pi 5) — two independent serial devices with real-time
communication needs, contending for the same bandwidth/power budget.

Fix: moved the accelerometer to a different USB controller than the
mainboard (a spare USB port on a separate bus — did not require a
powered hub). Confirm isolation with `lsusb -t`: the mainboard's `ch341`
device should be the only thing on its bus.

## Display Handler Bug Fix

A separate, unrelated bug: any time a genuine MCU error occurred, the
fork's custom `e3v3se_display.py` crashed while trying to *report* that
error, turning a potentially recoverable hiccup into a hard,
unrecoverable shutdown requiring manual `FIRMWARE_RESTART`.

```
TypeError: E3v3seDisplay.handle_mcu_error() takes 1 positional argument but 3 were given
Transition to shutdown state: Unhandled exception during run
```

Klipper's `klippy:notify_mcu_error` event fires with 2 extra arguments
(the error message and details) beyond `self`, but the handler was only
defined to accept `self`. Fix — accept (and ignore) the extra arguments,
since the handler already gets the error text a different way:

```python
# ~/klipper/klippy/extras/e3v3se_display.py, line ~665
# before:
def handle_mcu_error(self):
# after:
def handle_mcu_error(self, msg=None, details=None):
```

This is a genuine defect in the fork, not specific to this printer's
config — worth upstreaming.

## Load Cell Sensitivity

PRTouch's load cell (HX711 amplifier) is meaningfully more sensitive to
two things than a standard probe:

1. **Nozzle contamination** — even a thin film of residue on the tip
   changes the effective contact/trigger point.
2. **Heater PWM electrical noise** — the bed heater's rapid on/off
   switching to hold temperature can inject noise into the load cell's
   analog signal if wiring runs near the bed's power leads.

During one session, `PRTOUCH_PROBE_ZOFFSET` returned wildly inconsistent
results across consecutive runs (`1.517`, then `-2.340`, then `-2.530`,
then a clean `1.465` after cleaning the nozzle tip and specifically
running `TURN_OFF_HEATERS` immediately before probing). All four
calibration macros above bake in both fixes as standard practice. Always
sanity-check a new offset reading with `PRTOUCH_ACCURACY` (should land
around 0.005–0.02mm std dev) before trusting it enough to `SAVE_CONFIG` —
this probe type can be off by several millimeters on a bad reading, not
just a rounding error.

Also confirmed: bed thermal expansion measurably shifts the correct
Z-offset between room temp / PLA temp / ABS temp — another reason all
calibration is done hot, at actual print temperature, not cold.

## Lessons / Gotchas

- **`idle_timeout` (default 600s) can strand a paused print.** A
  filament-runout pause that takes longer than the idle timeout to
  resolve will silently disable steppers in the background, and Klipper
  will then correctly refuse `RESUME` with "Must home axis first" —
  because it genuinely no longer knows where the toolhead is. If the
  toolhead isn't confirmed clear of the print, **do not blindly `G28`** —
  a Z-home could crash straight into the part. Power off, manually
  reposition the toolhead clear of the print, power back on, then home.
  Increased to 18000s (5hr) here specifically to give real headroom for
  sourcing/loading filament mid-print without risking this again.
- **`SET_GCODE_OFFSET Z=<value>` is absolute, not additive** — pairing it
  with `Z_OFFSET_APPLY_PROBE` (which computes `new_offset = old_offset -
  gcode_offset`) can produce a nonsensical negative result if you're not
  careful about which command does what. Prefer editing `printer.cfg`
  directly for anything but small live babystep nudges
  (`SET_GCODE_OFFSET Z_ADJUST=0.01 MOVE=1`, which *is* additive).
- Fan miswiring (reversed polarity on a 2-pin connector) shows up as
  Klipper correctly commanding `speed: 1.0` while the fan physically
  doesn't spin — confirmable via
  `curl -s http://localhost/printer/objects/query?fan`.
- `PRINT_START` macros that call `BED_MESH_CALIBRATE` before the bed has
  actually reached and stabilized at target temperature (i.e. before
  `M190`, not after) will capture an inconsistent, in-between-temperature
  mesh every single print — reorder so the bed wait happens first.

## Tags

- **`stock-head-profile`** — full working config as of the pre-overhaul
  baseline: stock toolhead, PID tuned for the stock hotend, PLA PA 0.08,
  ABS Z-offset 1.465, bed shims installed. Use
  `git checkout stock-head-profile -- printer.cfg mcu.cfg` to pull this
  exact state back for comparison against the post-overhaul config.
