# Hardware TODO — encoder noise investigation

Logged 2026-07-14. Deferred on purpose — not addressing now, revisit before
re-enabling full quadrature (2-channel) position tracking.

## Motor / encoder reference

JGY-370 DC gearmotor with dual Hall-effect encoder:
- AB dual-phase incremental quadrature output, **11 pulses per revolution per
  channel**, 22-pole magnet ring.
- Encoder rated for up to 100kHz response — far above anything this project
  needs, so the encoder hardware itself is not the bottleneck.
- Wiring colors: red = motor+, black = motor-, green = sensor GND,
  blue = sensor VCC (4.5–24V), yellow = Signal A, white = Signal B.

## What we found

Built a throwaway diagnostic firmware (`EncoderDiagnostic.yaml`, still in the
repo, safe to reflash for retesting) that counts raw pulses on each Hall
channel independently, with no quadrature decoding at all. At the stated
~6000 RPM motor-shaft speed, the math predicts:

    11 pulses/rev x 6000 RPM = 66,000 pulses/min per channel (both channels,
    either direction, if the signal is clean)

Measured instead:
- **Down:** channel A ≈ 68,000/min (matches), channel B ≈ 104,000/min (~55% too high)
- **Up:** channel B ≈ 77,000/min (close), channel A ≈ 126,000/min (~90% too high)

So in both directions, one channel matches the true physical rate and the
other reports far more pulses than the rotation could produce — and *which*
channel is bad flips with direction. That's extra/spurious pulses being
injected onto the signal lines, not a resolution or component-capability
problem. This is what corrupted the position counter (integer overflow) when
we tried full quadrature decoding on both channels.

## Suspected cause

Wiring was done with UTP cable, pairs not matched to signal/return, soldered
directly to the motor (no connector). Motor power and Hall signal conductors
are very likely coupling PWM/commutation noise into the signal lines.

A hard reset under motor load was also observed once during testing — worth
re-checking after the cabling fixes below in case it recurs. If it does,
that's a separate power-supply-capacity issue, not encoder noise.

## Fixes to do, in priority order

1. **Rewire the UTP cable pairing.** Each signal should ride a twisted pair
   with its own dedicated return, not bundled arbitrarily with motor power.
   Suggested pairing for the 6 wires:
   - Pair 1: yellow (Signal A) + a GND return
   - Pair 2: white (Signal B) + a GND return
   - Pair 3: blue (sensor VCC) + green (sensor GND)
   - Keep red/black (motor power) on their own pair, physically separated
     from the signal pairs as much as the cable allows.

2. **RC snubber capacitor across the motor terminals** — ~100nF ceramic,
   soldered directly across red/black at the motor. Suppresses brush/
   commutation arcing noise at the source. Highest-leverage single fix,
   do this first.

3. **Optional — per-channel RC low-pass filter at the ESP8266 end**, only if
   noise persists after #1 and #2: ~1kΩ series resistor + ~100nF to GND on
   each Hall signal line. Real encoder edges from a slow-moving blind are
   well under 1kHz; PWM/commutation noise is much faster, so this filters
   noise without eating real pulses.

4. **Optional — local decoupling cap** (~100nF) across the Hall sensor's
   VCC/GND right at the motor, in addition to the snubber.

## Verification before re-enabling full quadrature

5. Reflash `EncoderDiagnostic.yaml` and rerun the same test: hold the motor
   at full speed in each direction, watch both channels in the logs
   (`esphome logs EncoderDiagnostic.yaml`). Confirm both converge to ~66,000
   pulses/min (±small margin) in *both* directions before trusting them
   together.

6. Once #5 passes, re-enable the `rotary_encoder` platform (both A + B) in
   `EspHomeScreenController.yaml`. Reconsider the D3/GPIO0 pin choice at that
   point too — it's a boot-strapping pin sharing circuitry with the onboard
   FLASH button, which is itself a plausible noise-pickup point independent
   of the cabling fix. If problems persist even with clean wiring, consider
   freeing a cleaner pin instead (e.g. disable UART logging to free up
   GPIO3/RX).

## Separate issue: unexplained resets / power stability

Independent of the encoder noise investigation above, the board has reset
itself unprompted multiple times this session — including once while sitting
completely idle (no motor activity) and reachable only moments before.
ESPHome's `safe_mode` boot-attempt counter climbed from 1 to 4 within about
two minutes at one point. This looks like a power-supply brownout/stability
issue distinct from the motor-noise problem, and needs its own investigation:

- Check what's actually powering the ESP8266 — same supply as the motor
  driver, or separate? What's the source (USB, wall adapter, buck converter)
  and its current rating?
- A motor's inrush/stall current sagging a shared or undersized rail is the
  most common cause of this pattern. Confirm with a multimeter (or scope) on
  the ESP8266's 3.3V rail while the motor starts/stalls.
- Since it also reset once at idle, also check for a marginal/noisy supply
  even without motor load (loose connection, bad regulator, etc.).
- Once cause is identified, likely fixes: separate supply for logic vs.
  motor, add bulk capacitance (e.g. 470–1000µF) near the ESP8266's power
  input, or a higher-current-rated supply.

## Current interim state

`EspHomeScreenController.yaml` is back on single-channel pulse counting
(D2/GPIO4 only, direction inferred from the firmware's own motor-direction
state, not from the encoder signal). Same architecture as the original
working proof-of-concept, with the calibration/position/jam-detection
feature set from the Test2.yaml port layered on top. D3/GPIO0 is currently
unused. This sidesteps the two-channel imbalance entirely, at the cost of
not cross-validating direction from the encoder itself.
