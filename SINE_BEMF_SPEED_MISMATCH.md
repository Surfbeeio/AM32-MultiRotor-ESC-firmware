# Sine/BEMF boundary speed mismatch

**Branch context:** `surfbee_v1.99`  
**Motor settings discussed:** `motor_kv = 420`, `motor_poles = 16`  
**Comparison case:** `motor_kv = 220`, `motor_poles = 16`

This note captures the speed mismatch between AM32's low-speed sine stepper region and the BEMF/six-step handoff assumptions.

---

## 1. Important distinction

Sine mode is effectively a commanded-speed open-loop stepper:

```c
advanceincrement();
step_delay = ...;
delayMicros(step_delay);
```

For each `step_delay`, the rotating electrical field has a fixed speed, and the rotor is expected to follow it.

BEMF/six-step mode is different: it applies commutation/duty and follows the rotor using BEMF timing. It is not commanding a fixed RPM in the same way.

However, when the firmware exits sine and hands back to BEMF, it seeds BEMF timing with a fixed assumed speed.

---

## 2. Current sine speed calculation

In the low-sine branch, current `v1.99` still uses:

```c
sine_target_step_delay = map(input,
                             48,
                             120,
                             7000 / motor_poles,
                             810 / motor_poles);
```

With `motor_poles = 16`, C integer division gives:

```text
7000 / 16 = 437 us per electrical degree
 810 / 16 =  50 us per electrical degree
```

There are 16 magnetic poles, so:

```text
pole_pairs = 16 / 2 = 8
```

Mechanical RPM from `step_delay` is:

```text
mechanical_RPM = 60,000,000 / (360 * step_delay_us * pole_pairs)
```

So the current low-sine commanded range is:

| Sine point | step_delay | Mechanical RPM, 16 poles |
|---|---:|---:|
| slow end | 437 us/deg | ~47.7 RPM |
| fast end | 50 us/deg | ~416.7 RPM |

So the low-sine branch currently spans roughly:

```text
~48 RPM to ~417 RPM mechanical
```

This range is independent of KV. KV affects torque constant and BEMF voltage per RPM, but not this commanded open-loop sine speed.

---

## 3. BEMF handoff seed speed

When sine exits back to BEMF, the firmware seeds BEMF timing with:

```c
commutation_interval = 9000;
average_interval = 9000;
INTERVAL_TIMER->CNT = 9000;
```

`commutation_interval` is in 0.5 us units for one 60 electrical degree sector.

Therefore:

```text
sector_time = 9000 * 0.5 us = 4500 us per 60 electrical degrees
per_degree = 4500 / 60 = 75 us per electrical degree
```

For 16 poles / 8 pole pairs:

```text
mechanical_RPM = 60,000,000 / (360 * 75 * 8)
               = ~277.8 RPM
```

There is also a forced sine-exit path:

```c
if(input > 200) {
    phase_A_position = 0;
    step_delay = 80;
}
```

`step_delay = 80 us/deg` gives:

```text
mechanical_RPM = 60,000,000 / (360 * 80 * 8)
               = ~260.4 RPM
```

So BEMF handoff is effectively expecting about:

```text
~260-278 RPM mechanical
```

---

## 4. The mismatch at the boundary

The sine branch runs while:

```c
input > 48 && input < 137
```

But the sine speed map reaches its fast end at:

```c
input = 120
```

So every value from:

```text
input = 120..136
```

is already clamped to the fastest low-sine speed.

That means at the BEMF->sine entry region around `input ~= 127`, sine is already targeting:

```text
~417 RPM mechanical
```

But the BEMF handoff/reacquisition assumptions are around:

```text
~260-278 RPM mechanical
```

Clear statement:

```text
BEMF -> sine:
    BEMF may be down near ~260-278 RPM,
    but sine target near the boundary is ~417 RPM.

sine -> BEMF:
    sine may be commanding ~417 RPM,
    but BEMF is initialized around ~260-278 RPM.
```

So the two modes do not meet at the same speed. Around the boundary, sine is asking for roughly:

```text
417 / 278 = 1.5x
```

or compared to the forced-exit seed:

```text
417 / 260 = 1.6x
```

This is effectively asking the loaded rotor to make a large speed change at the most fragile point in the control handoff.

---

## 5. KV comparison: 420 KV vs 220 KV

The commanded speeds above do **not** change with KV if `motor_poles = 16` remains the same.

| Point | step_delay | RPM, 16 poles | 420 KV BEMF approx | 220 KV BEMF approx |
|---|---:|---:|---:|---:|
| slow sine | 437 us/deg | ~47.7 RPM | ~0.11 V | ~0.22 V |
| forced exit seed | 80 us/deg | ~260.4 RPM | ~0.62 V | ~1.18 V |
| BEMF seed | 75 us/deg | ~277.8 RPM | ~0.66 V | ~1.26 V |
| current sine fast end | 50 us/deg | ~416.7 RPM | ~0.99 V | ~1.89 V |

Approximate BEMF voltage here is simply:

```text
V_bemf ~= mechanical_RPM / KV
```

This is only a rough motor constant comparison, not a measured phase voltage. It is still useful directionally:

- 220 KV gives about **1.9x more BEMF voltage per RPM** than 420 KV.
- 220 KV gives about **1.9x more torque per amp** than 420 KV, ignoring losses/saturation.
- But the open-loop sine commanded RPMs are the same for both if pole count is the same.

So a 220 KV / 16 pole motor should generally be easier for BEMF sensing and torque per amp, but it still sees the same sine/BEMF boundary speed mismatch.

---

## 6. Why changing the 15% crossover does not really fix this

Changing `sine_mode_changeover_thottle_level` changes the command-to-`input` remap.

It moves and stretches where external DShot/PWM lands on the internal `input` scale.

It does **not** change:

```text
sine slow step_delay = 7000 / motor_poles
sine fast step_delay = 810 / motor_poles
BEMF handoff seed = commutation_interval 9000
forced exit seed = step_delay 80
```

So changing the 15% setting mostly moves the boundary around in command space. It does not make sine and BEMF meet at the same speed.

---

## 7. Candidate fix: match sine fast end to BEMF seed

If we want the sine side to meet the BEMF side near the same speed, the fast end of the sine map should be closer to 75-80 us per electrical degree for 16 poles.

Current fast end:

```c
810 / motor_poles
```

For 16 poles:

```text
810 / 16 = 50 us/deg ~= 417 RPM
```

Candidate matched fast ends:

```text
1200 / 16 = 75 us/deg ~= 278 RPM
1280 / 16 = 80 us/deg ~= 260 RPM
```

So a direct experimental change would be:

```c
sine_target_step_delay = map(input,
                             48,
                             120,
                             7000 / motor_poles,
                             1200 / motor_poles);
```

or slightly slower/more conservative:

```c
sine_target_step_delay = map(input,
                             48,
                             120,
                             7000 / motor_poles,
                             1280 / motor_poles);
```

This would change the low-sine range for 16 poles to approximately:

| Fast-end constant | Fast step_delay | Fast low-sine RPM |
|---:|---:|---:|
| current 810 | 50 us/deg | ~417 RPM |
| proposed 1200 | 75 us/deg | ~278 RPM |
| proposed 1280 | 80 us/deg | ~260 RPM |

This directly targets the speed mismatch rather than only moving the command threshold.

---

## 8. Working hypothesis

The current handoff has a speed discontinuity:

```text
sine side near boundary ~= 417 RPM
BEMF side near boundary ~= 260-278 RPM
```

The loaded marine rotor may not tolerate that, especially in open-loop sine.

This explains why changing transition thresholds, phase seed, or slew alone may not fully solve the issue: the target speeds of the two modes still do not agree.

Next useful experiment:

1. Keep the current debug telemetry.
2. Change sine fast end from `810 / motor_poles` to `1200 / motor_poles` or `1280 / motor_poles`.
3. Run a slow ramp and the repeatable sweep.
4. Watch whether the stall disappears or moves.

If this improves the handoff, it confirms the mode-boundary speed mismatch is a primary contributor.
