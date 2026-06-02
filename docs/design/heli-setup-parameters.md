# Evora New-Heli setup — parameters → Rotorflight mapping

_Reference flow: VBar Control "Non-Mikado helis" setup wizard
(https://www.vstabi.info/en/node/1829). Evora collects the same parameters with
better screens, then writes the equivalent **Rotorflight** config over the link
via MSP. 2026-06-02._

## Principles (carried from the design law)
- **Nothing here touches the radio.** The radio output is the fixed Evora channel
  map (`AECR1T23`). Every parameter below is a **Rotorflight (FBL)** setting,
  written via MSP — same as VBar configures its flybarless unit. See
  `radio-is-a-controller.md`.
- **Verify every step, motor SAFE.** Blades off / motor disabled until the very
  end, exactly like VBar. Live steps (swash level, collective, cyclic, tail)
  require the heli bound + powered.
- **Confidence flags below:** ✅ = mapping is solid; ⚠️ = concept is right but the
  exact Rotorflight token/range must be verified against the bundled RF version
  (target **2.2.1**) before we write it. We do NOT ship a write path on a ⚠️ until
  verified against the RF source/CLI dump.

## Parameter map (VBar step → Evora collects → Rotorflight setting)

| # | VBar step | Evora collects (input) | Rotorflight target | Conf |
|---|-----------|------------------------|--------------------|------|
| 1 | Heli size | class chips: 450 / 550 / 700 / 800 / custom | apply an Evora **class preset** (PID profile + rates + filter baseline for that size) | ⚠️ need preset source |
| 2 | Load factory defaults | (action, folded into #1) | apply preset / `defaults`-from-base then overlay | ⚠️ |
| 3 | Sensor alignment | mounting chips: Flat / Inverted / Side + rotation (0/90/180/270) | board/gyro alignment: `align_board_roll/pitch/yaw` (a.k.a. board_align_*) | ⚠️ token |
| 4 | Swash plate type | chips: 120 / 135 / 140 / 90 / (mech/4-servo in Pro) | mixer **swashplate type** + servo geometry | ⚠️ token |
| 5 | Direction of rotation | toggle: CW / CCW (from above) | main rotor direction (drives torque/yaw comp + tail dir) | ⚠️ token |
| 6 | Leading/trailing edge | toggle: leading / trailing | swashplate **phase**/cyclic-ring orientation (servo cyclic direction) | ⚠️ confirm meaning |
| 7 | Swashplate leveling | **live**: nudge AIL/ELE/COL to level swash, center throw | per-servo **center/trim** (`servo <n> mid`/trim) + collective/cyclic neutral | ⚠️ token |
| 8 | Min/Max collective | **live**: set max collective pitch, target ±12–14° (pitch gauge) | collective **range/travel** limits (deg → servo throw) | ⚠️ token |
| 9 | Cyclic calibration | **live**: set cyclic = 8° (blade over boom) | **cyclic ring / cyclic deflection** limits | ⚠️ token |
| 10 | Tail servo type | center pulse 760/1520 µs + frame rate (analog/digital Hz) | tail/yaw **servo type**: center pulse + update rate | ⚠️ token |
| 11 | Tail servo calibration | **live**: tail endpoints (start safe 40 → expand to mech limit) | yaw servo **min/max** + tail pitch limits | ⚠️ token |
| 12 | Governor type | chips: **External (ESC)** / **Rotorflight e-gov** / (Nitro = Pro/out-of-scope) | `gov_mode` = PASSTHROUGH (external) / STANDARD-or-MODE (e-gov) | ⚠️ enum |
| 13a | External gov | (nothing — passthrough; optional ESC notes) | `gov_mode = PASSTHROUGH`; Evora just passes the throttle channel | ✅ concept |
| 13b | e-Governor | gear ratio (number); motor **pole count** (magnets = poles/2, e.g. 5 ↔ 10-pole); headspeed per bank | `gov_gear_ratio`, `motor_pole_count`/RPM source, `gov_headspeed` per profile | ⚠️ token |

## Evora-specific additions (not in the VBar non-Mikado flow)
- **Bind / connect** step (our LINK): bind ELRS + auto-detect Rotorflight + read
  back current config, before any writes. ✅
- **Banks** (Evora flight modes): headspeed + rate feel per bank (3 banks) →
  Rotorflight **rate/PID profiles** + per-profile governor headspeed. ⚠️ map to
  profile switching on AUX2 (already in the channel map).
- The fixed channel map + mode activation conditions (arm/rescue/profile/etc.)
  are baked into the Evora Rotorflight preset already — the wizard does NOT ask
  for these (radio is a controller).

## Proposed Evora wizard structure (revised from the placeholder 12)
1. **Before we begin** — blades off, motor safe, servo arms ~90°.  [no write]
2. **Pick your heli** — class/size → load preset.
3. **Bind your receiver** — ELRS bind + detect Rotorflight.
4. **Board orientation** — mounting + rotation → alignment.
5. **Swashplate type** — geometry.
6. **Rotor direction** — CW/CCW.
7. **Swash phase** — leading/trailing edge (+ live swash-direction check).
8. **Level the swash** — live, sets servo trims/centers.
9. **Collective range** — live, set max pitch (±12–14°).
10. **Cyclic range** — live, set 8°.
11. **Tail servo** — type (center pulse + rate).
12. **Tail travel** — live endpoints.
13. **Governor** — type (external / e-gov).
14. **Governor detail** — gear ratio + motor poles (e-gov only).
15. **Banks & head speed** — per-bank headspeed + feel.
16. **Review & write** — push to Rotorflight via MSP → "Ready to fly".

Steps 8–12 + 15 are the "live verify" screens (need the heli connected); the
rest can be set from selections. Each keeps the MOTOR-SAFE banner.

## Open questions for the owner (heli expert)
1. **Class presets:** where do the per-size defaults come from — do you have
   known-good Rotorflight dumps per size (450/550/700/800), or should Evora
   start from RF defaults + a few overlays? This is the biggest unknown.
2. **Leading/trailing edge (#6):** confirm this is the swash **phase**/servo
   cyclic-direction concept in Rotorflight terms (vs. something else).
3. **Governor scope:** support External (passthrough) + Rotorflight e-governor;
   treat **Nitro** as out-of-scope (electric-only)? 
4. **Live steps:** confirm we do swash-level / collective / cyclic / tail
   **live on the heli** (like VBar), not just write computed defaults.
5. Anything VBar collects that you want to **drop** for v1, or anything missing?

## Next step
Once the mapping + structure are confirmed (and I verify the ⚠️ tokens against
Rotorflight 2.2.1's CLI/MSP), I rebuild the wizard with these steps in the Evora
aesthetic at both resolutions, then we wire the MSP writes when the heli's on the
bench.
