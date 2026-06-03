# Evora menu system — design

_Created 2026-06-03. Owner-approved: Pro = a toggle in System; build full screen shells now, wire
Rotorflight MSP later. Develop on the TX16S; the single EV_SC codebase renders MK3 (800×480) for free._

## Principles
- **Ground-up, not an EdgeTX skin.** Evora screens are bespoke (like `evora_home`/`evora_wizard`);
  we reference EdgeTX only for *how the plumbing works* (read/write `g_eeGeneral`, calibration, ADC).
- **Radio is a controller.** Basic mode shows **no flight settings** — the radio is a fixed universal
  controller; flight tuning lives in Rotorflight. **Pro** is an opt-in area that *does* surface
  Rotorflight tuning (over MSP), for people who want the full surface. One clear gate.
- **Intuitive > complete.** Progressive disclosure: simple by default, expert on demand.

## Navigation model
- **Page stack within a destination.** Today each destination (Helicopters/System/Bind/Tools) is a
  flat one-level overlay. Add a lightweight **push/pop stack**: a row can open a **sub-page** with its
  own header + Back, Back pops to the parent (not all the way home). Home tabs still open the
  top-level destinations.
- **Scrollable pages.** A reusable `listPage` container with `LV_OBJ_FLAG_SCROLLABLE` set; content
  taller than the panel scrolls by **touch drag/flick** (native LVGL). Long Settings/Pro lists use it.
- **Encoder (scroll wheel).** Wire the rotary encoder to the LVGL input group so the wheel scrolls
  lists and edits the focused value (also satisfies the standing "scroll wheel adjusts trim" request).
  *Deferred slightly:* can't be verified in the screenshot sim (encoder≠touch) — needs hardware. Touch
  nav ships first; encoder wiring lands alongside, validated on the 16S.

## Screen tree
```
Home (idle | flight)
├─ HELICOPTERS  → heli list → [heli] details · + Pair → BIND
├─ BIND         → search/pair state machine (built)
├─ SYSTEM       → device settings (radio-local, fully functional now)
│   ├─ Stick calibration        (launch real calibration)
│   ├─ Sounds / volume           (g_eeGeneral)
│   ├─ Backlight                 (g_eeGeneral)
│   ├─ Units                     (g_eeGeneral)
│   ├─ Pro mode  [ OFF | ON ]    ← the gate
│   └─ About Evora               (version/build/credits)
│   └─(Pro ON)─ PRO TUNING ▸     ← appears only when Pro is on
├─ TOOLS        → Blackbox · Servo test · Receiver/Link info · Channel monitor
└─ PRO TUNING (shells now, MSP later)
    ├─ Profiles / banks      (1·2·3 switch + per-bank values)
    ├─ PIDs                  (P/I/D per axis)
    ├─ Rates                 (rate/expo per axis)
    ├─ Filters               (gyro/D-term/notch)
    ├─ Governor              (full gov params)
    └─ Blackbox tuning       (Phase 5)
```

## Build state per the owner decision
- **Fully functional now:** navigation, scrolling, System device settings (sound/backlight/units read
  & write `g_eeGeneral`; calibration launches; About). Tools info pages. Pro toggle (persists a flag).
- **Shells now, dormant until MSP:** all PRO TUNING screens — navigable, scrollable, laid out, with
  placeholder values; live read/write to Rotorflight wired when MSP lands (heli on the bench).

## Reusable components (added to evora_screen)
- `listPage(title)` — scrollable page with header + Back, returns the scroll container.
- `navRow(parent, label, value, onTap)` — tappable label/value row with a chevron + hairline; opening
  a sub-page pushes onto the stack.
- `toggleRow(parent, label, bool*, onChange)` — on/off switch row (used by Pro mode).
- Editors: `stepperPage` (numeric ± with range), `choicePage` (pick-one list) — reused across settings.

## Verification
Build + screenshot each new screen in the 480 sim (`./sim.sh` / capture sweep); sweep 800 to confirm
EV_SC scaling. Encoder behavior verified on the 16S when wired.
