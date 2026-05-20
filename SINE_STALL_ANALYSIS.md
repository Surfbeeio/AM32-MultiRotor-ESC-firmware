# Sine-mode stall investigation — working notes

**Branch:** `surfbee_v1.99`  ·  **MCU:** STM32G071 (64 MHz)  ·  **Date:** 2026-05-20
**App:** Surfbee (BLDC + prop, run in water — i.e. loaded)

This document captures the analysis of the low-speed **sine-mode stall**: the motor
drops/stalls at low throttle in the sine (crawl) region. It records what we
confirmed, what turned out to be unreliable, the relevant code, and the levers.

---

## 1. Hardware / configuration context

| Item | Value | Source |
|---|---|---|
| MCU | STM32G071, 64 MHz | `Inc/targets.h` (MCU_G071) |
| `TIM1_AUTORELOAD` (ARR) | **2667** → ~24 kHz PWM default | `Inc/targets.h` |
| Throttle input | **DShot** (no servo deadband — `adjusted_input = newinput`, `main.c:1961`) | — |
| Throttle calibration | idle/min ≈ **1500 µs**, nonstandard | user |
| Sine→six-step changeover (param 40) | **15%** | EEPROM `[40]`, default in src = 5 |
| `sine_mode_power` (param 45) | **10** (maxed) | EEPROM `[45]`, default in src = 5 |
| `SINE_DIVIDER` | **2** | `Inc/targets.h:441` (compile-time) |
| `DEBUG_MOTOR_STATE` | **enabled** | `main.c:209` |

> ⚠️ With `DEBUG_MOTOR_STATE` on, the serial-telem **consumed-mAh field is overloaded
> with a state code** and real mAh is NOT reported. Disable for a release build.

### Telemetry decode (DEBUG_MOTOR_STATE), `main.c:1414-1448`
The `esc_mah` field encodes motor state instead of mAh:
- **0 / 1000** = not armed / armed-stopped
- **2000** = running in **BEMF / six-step**
- **3000 + s** = running in **sine**, `s` = per-cycle current swing (0–999). ~3000 locked, climbing → slipping.
- **4000 + N** = **stuck-rotor cut fired**, `N` = commutations since last sine→BEMF handoff.

`eRPM` telem also carries fault sentinels (0xFFFE = stuck-rotor) — `main.c:1397-1410`.

---

## 2. Root-cause model (current best understanding)

The stall is **open-loop sine losing the loaded rotor**:

- Sine drive is **fully open-loop**. The field marches at a throttle-derived rate
  regardless of what the rotor does.
- As throttle pushes the commanded field speed up (and/or under prop load), the
  rotor can't follow → it **slips and stalls** while still in sine.
- This is invisible to both protections (see §5), so it's only ever *detected*
  later — if/when it hands to BEMF against a dead rotor (`4000+N`, N≈0).

**Six-step / BEMF works fine** — in every log, when BEMF engaged it spun the rotor
up to 342–428 eRPM. The weak link is specifically blind, low-amplitude sine.

---

## 3. Key code locations

| What | Where |
|---|---|
| Sine vs six-step decision (`input < 137` = sine) | `main.c:2156` |
| Sine→six-step exit ramp (input ≥ 137) | `main.c:2200-2229` |
| Throttle→`input` map (uses param 40) | `main.c:2012-2016` |
| Sine entry seed: field speed from commutation interval | `main.c:1266-1285` |
| Sine entry seed: precise rotor angle | `main.c:1233-1250` |
| Field-speed slew (`SINE_SLEW_RATE`) | `main.c:2168-2171` |
| Amplitude soft-engage (`SINE_POWER_START/SLEW`) | `main.c:2172-2181` |
| In-sine current-swing probe | `main.c:2182-2196` |
| **eRPM in sine = `600/step_delay` (COMMAND, not feedback)** | `main.c:2198` |
| eRPM in six-step = measured from commutation timing | `main.c:2055` |
| **Sine amplitude write (CCR)** | `main.c:1544-1546` |
| Sine lookup table `pwmSin[]` | `main.c:503-547` |
| Tuning constants (`SINE_SLEW_RATE=2`, `SINE_POWER_START=2`, `SINE_POWER_SLEW=2`) | `main.c:296,308-309` |

### v1.98 / v1.99 attempts (git history)
- `0e535e1` synchronized sine handoff (seed step rate) — **reverted** (no-op at boundary)
- `70eaa3e` defer handover until rotor slows — **reverted**
- `f0bedad` slew field speed + matched entry seed + 4000+N fault counter — kept
- `340583f` seed precise rotor angle on entry — kept
- `22d2ff1` amplitude soft-engage + slip probe — kept
- `0a03d2d` per-cycle current-swing probe (replaced fixed-angle probe) — kept
- `a574e15` doubled amplitude ramp time — kept

All four open-loop levers (field speed, entry angle, handoff timing, amplitude)
have been touched. Stall persists.

---

## 4. The crossover point

- In firmware: the line is a single hard compare `input = 137` (`main.c:2156`),
  **no hysteresis**. Below = sine, at/above = six-step.
- Back-mapping `input=137` through `main.c:2012-2016` at param 40 = 15 →
  `adjusted_input ≈ 245/2047 ≈ 12%`. (At 15% the map already yields input=160.)
- In µs (assuming 1500–2000 linear): actual handoff ≈ **~1560 µs**, configured
  15% ≈ 1575 µs.

> ⚠️ **The µs numbers are NOT confirmed by the logs.** All stall/sweep tests were
> done with a ~2 Hz throttle sweep, and the command→ESC→telemetry pipeline lag
> (~100–250 ms) is a large fraction of that period. Cross-correlation of commanded
> throttle vs measured rpm gave only ~0.36 (weak, broad, no clean lag), so the
> throttle axis can't be trusted to locate a µs threshold. **A slow single ramp
> (creep up over ~8–10 s, hold, creep down) is required to pin the crossover.**

---

## 5. Why neither protection catches the sine stall

- **Stuck-rotor cut (`4000+N`)** relies on the BEMF timeout, which only counts in
  six-step. A rotor dying *in sine* is invisible until handoff — and at handoff
  BEMF usually catches it, so the cut never arms. **Zero `4000` events in any log.**
- **In-sine current-swing probe** stayed moderate (max ~100/999) even while the
  rotor collapsed — it did not flag the stalls. Effectively blind here.
- **eRPM is NOT feedback in sine** (`main.c:2198`, `e_rpm = 600/step_delay`) — it's
  the commanded field speed echoed back. A spinning and a stalled rotor report the
  same eRPM in sine.
- **Conclusion accepted with the user:** sine has *no usable feedback* (current
  probe also deemed pointless). Sine is permanently open-loop/blind; design around it.

---

## 6. Sine table verification — CLEAN

`pwmSin[]` (`main.c:503-547`) was checked entry-by-entry against an ideal sine:
- 360 entries exactly; range 0–360; key points dead-on (idx0=180, idx90=360, idx180=180, idx270=0).
- Worst deviation from ideal = **0.5** (integer rounding); monotonic in all three segments.
- **No typo.** The waveform is not the problem.

---

## 7. Sine amplitude math — the torque ceiling

Active CCR write (non-GIMBAL), `main.c:1544`:
```c
TIM1->CCR1 = (((2*pwmSin[pos]/SINE_DIVIDER)+gate_drive_offset)*TIMER1_MAX_ARR/2000)*sine_power_ramp/10;
```
Computed exactly (G071: ARR=2667, SINE_DIVIDER=2, DEAD_TIME≈45). The line is sound —
no overflow, no clipping, never negative. But the **output amplitude is low**:

| sine_mode_power | SINE_DIVIDER | peak phase duty | phase-to-phase pp |
|---|---|---|---|
| 5 (src default) | 2 | 10.1% | 9% |
| **10 (current config)** | **2** | **20.2%** | **18%** |
| 10 | **1** | **38.2%** | **36%** |

- At the **current setting (power=10, divider=2) the sine peaks at only ~20% phase
  duty** — and it still stalls under load. So ~20% is not enough torque.
- `sine_mode_power` is **already maxed (10)** → the only remaining amplitude lever is
  **`SINE_DIVIDER`** (`Inc/targets.h:441`): setting **2 → 1 roughly doubles the drive**
  (peak 20%→38%), no clipping (CCR 1020 of 2667), with headroom to spare.
- **PWM frequency does NOT affect sine amplitude** — duty % is invariant to ARR
  (24 kHz and 48 kHz give identical %). PWM freq only changes switching loss, ripple,
  audible noise, BEMF sampling.
- **Heat caveat:** in the sine band back-EMF ≈ 0, so phase current ≈ V/R. Doubling
  the voltage ~doubles current and ~quadruples I²R heating. Measured sine current was
  <~1 A, so even 4× is a few amps — fine for transit, watch temp for sustained crawl.

---

## 8. What the logs do / don't show

**Trustworthy (phase-independent):**
- Rotor eRPM pulsed hard (~14 ↔ ~171) during sweeps.
- BEMF, when engaged, spun the rotor to 342–428 eRPM (six-step healthy).
- No `4000` stuck-cut ever fired.
- Swing metric stayed moderate throughout (never flagged the stall).
- The final stall ended with the rotor wound down to ~14 eRPM (stopped).

**NOT trustworthy (corrupted by 2 Hz sweep + pipeline lag):**
- Any specific µs for the crossover.
- The "rpm collapses as throttle rises" correlation (artifact; overall corr was
  actually slightly positive).

Logs reviewed: `mav_rc_log_2026-05-20T05-52-54`, `06-29-43`, `06-38-24`, `06-40-02`
(throttle commanded via MAVLink RC_OVERRIDE, ~2 Hz sine sweeps, center ~1529 / amp ~28).

---

## 9. Levers & strategy

Since sine is open-loop with no feedback, the only defenses are:
1. **Keep commanded field speed in a band the loaded rotor can follow** (low).
2. **Enough amplitude/torque margin that it can't pull out of sync** → `SINE_DIVIDER` (2→1),
   `sine_mode_power` (already maxed).
3. **Slew** the field so it can't accelerate faster than the rotor (`SINE_SLEW_RATE`).
4. **Hand to BEMF early** (lower param 40 / add hysteresis) — BEMF has feedback and works.

**Strategy depends on one open question:** does the craft need to *cruise continuously*
at low speed in the sine band, or only *pass through* it on the way up/down?
- Pass-through → minimize sine, hand to BEMF as early as it can hold.
- Continuous low-speed → must brute-force amplitude (SINE_DIVIDER=1) since there's no feedback.

---

## 10. Next steps / open items

- [ ] **Slow-ramp test** (not a fast sweep) to pin the real crossover µs and see slip cleanly.
- [ ] Decide: cruise-in-sine vs pass-through (drives the whole strategy).
- [ ] Try **`SINE_DIVIDER = 1`** (compile-time; rebuild + reflash) — biggest available torque bump.
- [ ] Consider **hysteresis** on the `input=137` changeover (`main.c:2156`) to stop boundary chatter.
- [ ] Confirm exact throttle calibration (servo min/max) + param 40 to convert `input=137` to exact µs.
- [ ] Remember to disable `DEBUG_MOTOR_STATE` for any release build.
