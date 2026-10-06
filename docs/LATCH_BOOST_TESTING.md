# DC latch boost testing (ESP32 / ESP32-C5 and ESP8266)

## What to look for

Build with `-DENABLE_DEBUG`, open the serial monitor (115200 baud), and run a
zone. Read the existing `boost on voltage:` / `boost off voltage:` lines and
the new messages:

```text
[LATCH] boost target=9V start=1000mV
[LATCH] boost end=9200mV after 120ms (fallback=0)
```

- A start value already near or above the target, or a target reached in about
  20–25 ms, suggests the sense pin is not measuring capacitor voltage or the
  divider/scaling is wrong. Confirm with a physical voltage measurement.
- An end value well below the target after the full `tmax` means more charging
  time may be needed: increase **DC boost time** (`bst`).
- `fallback=1` means the sense reading rose by less than 1000 mV and time was
  still available. The controller finishes charging in timed mode, but check
  the hardware sense path. The logged end value is read again after fallback.

The time limit is `tmax = bst × 4` ms. Regulated boosting always lasts at least
20 ms, even if `tmax` is shorter or the initial reading already meets the target.
`laton=0` / `latof=0` means the default **9 V**, not timed mode. Out-of-range
voltage values retain the existing timed behavior.

## How to test

1. Run each zone with both latch voltage options (`laton` / `latof`) set to
   **0 (=9 V), 12, and 24 V**. Stay within the valve and board voltage ratings.
2. Measure capacitor voltage with a multimeter or oscilloscope and compare it
   with the logged end value. A scope is preferable for short charge/pulse
   events; measure before the output pulse discharges the capacitor.
3. Try DC boost times of **320, 600, and 1000 ms** (`bst` values **80, 150, and
   250**, respectively).
4. Confirm that every zone opens and closes, repeating about **20 times** for
   each setting. Allow the normal controller sequencing between operations.
5. Check board divider resistor values against `LATCH_VSENSE_R_TOP_x10` (798)
   and `LATCH_VSENSE_R_BOT_x10` (15), representing the supplied 79.8k/1.5k
   scaling. These macros can be overridden at build time and must match your
   board schematic. Check that `PIN_LATCH_VOLT_SENSE` actually measures the
   capacitor; firmware uses `PIN_CURR_SENSE` if no dedicated pin is defined.
   ESP32 uses calibrated ADC millivolts; ESP8266 retains its 10-bit, 1 V ADC
   scaling. Shared current-sense wiring needs particular attention on C5.
6. Regression check: confirm **AC/DC (non-latch)** controllers behave as before.

The output pulse remains unchanged (150 ms on the v2 latch path). No persistent
option layout or firmware version changes are required by this fix. Physical
voltage measurements and repeated switching are necessary to verify hardware;
debug logs alone cannot establish that the sense circuit is wired correctly.
