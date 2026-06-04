# Evora — current status & how to resume

_Last updated: 2026-06-03_

Evora is an open-source, **firmware-level** recreation of the Mikado VBar Control Touch
*experience* for RC helis: bind + a guided on-radio wizard + fly, no PC. See [`../README.md`](../README.md)
for the vision and [`superpowers/specs/2026-06-01-evora-architecture-design.md`](superpowers/specs/2026-06-01-evora-architecture-design.md)
for the full architecture.

## North Star (confirmed by owner)
Evora is its **own bespoke OS with a gorgeous interface**. We keep EdgeTX's *engine* (gimbal/ADC
drivers, RF/CRSF, mixer, failsafe, storage, bootloader — the dangerous, proven plumbing) and **rip
out EdgeTX's entire GUI**, replacing it with Evora's own (boot, home, navigation, wizard). Not a
widget/skin inside EdgeTX. Visual language = **"aerospace instrument, blacked-out"**: layered dark
backgrounds (navy `#121c28`), Rajdhani instrument numerals + Barlow labels, **amber `#ff9242`** accent +
**green `#3fe0a0`** = healthy, circular gauges, gradient cards, hairlines. (A BRUTALIST re-skin —
black + lime/orange/red/white, Archivo Black, big numbers — was built and **reverted 2026-06-03** at
owner's call; preserved on `Evora-TX@brutalist-theme` + `mockups/gallery/` if ever wanted again.)
Flight performance stays Rotorflight's (untouched FC). **The radio is a fixed
universal controller** — no radio settings; all flight layering lives in Rotorflight (see
`design/radio-is-a-controller.md`).

## Two supported radios (only these, for the foreseeable future)
| | Evora TX (TX16S) | Evora TX (MK3) |
|---|---|---|
| MCU | STM32F407 | STM32H750 (H7, SDRAM/QSPI) |
| Panel | 480×272 | 800×480 |
| EdgeTX flavor | `tx16s` (PCB=X10/TX16S) | `tx16smk3` (PCB=TX16SMK3) |
| CI artifact | `evora-tx16s` | `evora-tx16smk3` |

One codebase serves both: the UI is authored in a **logical 480×272 space** and the `EV_SC(v)` macro
(`mainview/evora_fonts.h`) scales every literal coordinate to the panel (1.0× / ~1.667×). Bottom bars
anchor to the real edge via `height()`. Per-resolution font + dog-image sets (`fonts/evora` /
`fonts/evora-lrg`, identical symbol names) are selected by `BITMAPS_DIR` in `fonts/CMakeLists.txt`.

## Architecture
- **Evora TX** = fork of **EdgeTX** (`ScottThomasGraham/Evora-TX`, baseline `v2.12.1`, branch `evora`).
- **Evora Link** = fork of **ExpressLRS** (`ScottThomasGraham/Evora-Link`, baseline **`3.6.3`**, branch `evora`) — both TX module + RX; later a private high-bandwidth config channel. (Brand only so far; ELRS build is a Phase-0 task.) Re-pinned `4.0.1 → 3.6.3` on 2026-06-02 (ELRS 4.0 breaking changes) — see [`upstream-baselines.md`](upstream-baselines.md).
- **Rotorflight** = untouched FC, configured over the link via MSP.
- Meta repo: `ScottThomasGraham/Evora` (branch `main`) — docs, mockups, design.

## What's built (real firmware, both radios, CI-green, emulator-verified)
- **Boot splash** — full-screen SD `/IMAGES/splash.png` (per-radio asset under `docs/sdcard/<radio>/IMAGES/`) with the dog+EVORA hero; embedded `splash_logo.png` (per-res, RGBA) is the no-SD fallback.
- **Two-state home** — idle radio-info ↔ flight dashboard (circular `lv_arc` headspeed gauge + battery hero + mini-stats), switches on `TELEMETRY_STREAMING()`. Faint dog watermark + amber dog corner-glyph. New model (wizard) + Edit model CTAs on the home.
- **Destination screens** — Helicopters / System ("no flight settings here") / Bind / Tools (touch + back).
- **13-step New Heli Setup wizard** — the real flow, MOTOR-safe, verify-every-step. Steps: Before we begin · Bind · **Servo pulse (760/1520)** · Pick heli · Board orientation · Swash type · Rotor direction · **Swash check (live)** · **Collective pitch (live)** · **Tail travel (live)** · Governor (ESC/Rotorflight/Nitro) · Governor settings · Ready to fly. Diagrams are **drawn with LVGL primitives** (rings/arcs/lines/labels — ~0 flash, scale via EV_SC). Interactions are real (select/segmented/live ±/governor branch); **MSP writes are stubbed** until the heli's on the bench.
- **Real-firmware overviews:** `docs/emulator/overview-tx16s.png` + `overview-mk3.png` (every screen). Per-screen: `fw-*.png` / `mk3-*.png`; wizard steps `fw-wiz-*.png` / `mk3-wiz-*.png`.
- **Custom fonts** generated via lv_font_conv (Rajdhani Bold/SemiBold + Barlow Semi-Condensed) in `radio/src/fonts/evora{,-lrg}/`, declared in `mainview/evora_fonts.h` (EVF_* macros). Fits F407 flash. (`tools/fonts/regen.sh` + the Archivo/Plex TTFs remain in the meta repo for the shelved Brutalist theme.)
- All telemetry/setup **values are DEMO placeholders** — live data needs the heli.

### UI source files (`forks/evora-tx/radio/src/gui/colorlcd/`)
- `mainview/evora_home.{h,cpp}` — two-state home (buildIdle/buildFlight, dog images, gauge, watermark).
- `mainview/evora_screen.{h,cpp}` — Helicopters/System/Bind/Tools overlays.
- `mainview/evora_wizard.{h,cpp}` — the 13-step wizard. `STEPS[]` (section/title/sub/live/finish) + `switch(step)` body; LVGL-primitive diagram helpers (`poly` via a static point pool, `ring`/`dot`/`arcSeg`/`box`/`card`); governor branch on `sel[10]`; live ± via `adjust()`. Header labels are TOP-aligned (NOT `*_MID` against the full screen).
- `mainview/evora_fonts.h` — `EV_SC` scale macro, EVF_* font macros, dog image externs.
- `mainview/layout.cpp` — makes EvoraHome the main view; clears EdgeTX topbar.
- `radio/src/fonts/{evora,evora-lrg}/` — fonts + dog images. `radio/src/bitmaps/{480x272,800x480}/splash_logo.png`.

### Emulator preview toggles (ship as default; flip only to screenshot)
`evora_home.cpp`: `EV_FORCE_STATE` (0 auto / 1 flight / 2 idle), `EV_PREVIEW_SCREEN` (-1..3), `EV_PREVIEW_WIZARD` (0/1). `evora_wizard.cpp`: `EV_WIZARD_START` (0..12).

## How to build  →  see [`build-and-simulator.md`](build-and-simulator.md)
Local builds now work natively on this Mac (the old qemu-segfault blocker is solved). In short, from
`forks/evora-tx/`:
- **Flashable firmware:** `./build-evora-tx.sh` → `build-tx16s-native/arm-none-eabi/firmware.bin` (+ `dist/`).
  Native arm64 toolchain image (`evora-tx-native:14.2`), no emulation ICEs. MK3 local build = TODO.
- **Simulator (fast UI loop):** `./sim.sh` → `sim-shot.png`. Verifies anything **visual** in seconds;
  **can't** verify audio or rotary-encoder input (touch sim — test those on hardware).
- CI still builds both radios on push to `evora` (`gh run download … -n evora-tx16s`); use it for the MK3 artifact.

The full toolchain rationale, preview hooks, and sim caveats live in
[`build-and-simulator.md`](build-and-simulator.md). Upstream pins (EdgeTX/ELRS/Rotorflight) + reference
repos: [`upstream-baselines.md`](upstream-baselines.md).

## Next steps
1. **(needs heli on bench)** Wire the wizard's **MSP writes** + live telemetry/servo motion (the steps are collect-only today); live data in home + flight dashboards.
2. **Class presets** — owner supplies per-size Rotorflight files/offsets from flight testing; "Pick your heli" applies them. Bake the standard `AECR1T23` rc_map + mode-activation preset into the FlyDragon.
3. **Evora Link (ELRS):** build TX (`Unified_ESP32_2400_TX_via_ETX`) + author the FlyDragon RX hardware layout (`Unified_ESP8285_2400_RX_via_BetaflightPassthrough`); pick a binding phrase; then the private fast-config channel.
4. **Phase 0 hardware (owner):** flash `evora-tx16s` (bootloader-recoverable) once the SD reader arrives; copy `docs/sdcard/tx16s/IMAGES/splash.png` to the SD. MK3 when on hand.

## Session log
**2026-06-03 (perf) — DMA2D acceleration + render simplification (fast UI pass).**
- Owner flagged sluggish tile-drag + menu-open. Root cause: LVGL was software-rendering
  gradients/alpha on every frame, and the menu rebuilt ~80 objects on each open. The
  **TX16S is an STM32F429** (`CPU_TYPE_FULL STM32F429xI`, `-DSTM32F429xx -DSDRAM`) — it
  **has DMA2D (Chrom-ART) + LTDC + SDRAM**, but LVGL's DMA2D was disabled.
- **Enabled `LV_USE_GPU_STM32_DMA2D`** (lv_conf.h), gated to DMA2D-capable targets
  (F429/F439/H7) and never the simulator or F407 colorlcd radios (T16/T18). EdgeTX's
  `!LV_USE_GPU_STM32_DMA2D` guards (lcd.cpp/boot_lcd.cpp/dma2d.cpp) switch its own DMA2D
  blitter + software path off → no peripheral contention. **Firmware links clean; sim
  still builds with it gated off.** Rendering correctness is **hardware-verify only**
  (sim can't exercise DMA2D) — revert the one flag if artifacts appear.
- **Flat fills** (solid card/background, opaque borders — no per-pixel gradient/alpha) +
  **build-once menu** (hidden, show/hide + slide; no per-tap construction) + bigger MENU tab.
- Bin: `evora-tx16s.bin` (DMA2D build). `Evora-TX@03041e2f2`.

**2026-06-03 (late) — VBar-style sectioned right-side menu + Tuning pages.**
- Built the **VBar Control Touch menu structure** in the instrument theme (`EvoraHome::openMenu/
  closeMenu/menuTab`): slide-in panel from the right over the dimmed home, amber edge tab, scrim/X to
  close. **Data-driven sections** — MODEL (New/Edit/Switch/Bind) · TUNING (Main/Tail/ESC/Governor) ·
  MONITOR · RADIO · TOOLS — each row an `LV_SYMBOL` icon + label + desc. **Context-gating** (MONITOR
  only when connected) + **`expert` markers** (progressive disclosure).
- **Three-layer model:** New model = wizard (+ erase-&-start-fresh confirm, keeps bind) · Edit model =
  set-once geometry by discipline (main/tail/ESC) · **Tuning = the feel layer** (4 new `PG_TUNE_*`
  pages via `EvoraScreen::openTune`: quick PID + stick-feel chips, tap-to-adjust).
- Spec: [`superpowers/specs/2026-06-03-vbar-menu-design.md`]. Mockup + sim renders in `mockups/gallery/`
  (`menu-vbar`, `sim-menu`, `sim-tune-main`). Bin: `evora-tx16s.bin`. **MSP writes still stubbed.**
  TODO: wire MONITOR screens (currently rows only); the actual RF-default load on reset; trim the
  redundant bottom toolbar now that the menu is primary nav.

**2026-06-03 (night) — REVERTED Brutalist; back to the instrument theme.**
- After living with the full Brutalist build, owner preferred the original **amber/teal "aerospace
  instrument"** look. Reverted `Evora-TX` `evora` to **`cdedbd508`** (the pre-Brutalist tip) — this
  restores the instrument theme **and keeps** the Edit-Model discipline **tiles + horizontal scroll**
  and the **per-discipline Pro mode** (all built before the re-skin). The **right-side drawer + two-tone
  wordmark + giant-number flight + watermarks came off** with the theme (bundled in the Brutalist commit).
- **Nothing lost:** Brutalist is preserved on branch **`Evora-TX@brutalist-theme`** (fonts, palette, drawer,
  pull-tab) + `mockups/gallery/` (concepts, boards, `board-06x` color system) + memory `[[evora-design-language]]`.
  To resurrect: `git merge brutalist-theme` (or cherry-pick the drawer commit) and rebuild.
- Bin rebuilt from `cdedbd508` = the instrument look back on hardware.

**2026-06-03 (eve) — BRUTALIST re-skin (SHELVED — see revert above) + right-side app drawer.**
- After a 12-concept gallery + finalist boards (`mockups/gallery/`), owner **locked BRUTALIST** with a
  functional color system: true black, **LIME** primary/live · **ORANGE** pro/advanced · **RED** danger ·
  **WHITE** dog/brand. Canonical reference `mockups/gallery/board-06x.html`. Memory: `[[evora-design-language]]`.
- **Fonts swapped** to **Archivo Black** (display/slabs) + **IBM Plex Mono** (data) via `npx lv_font_conv`
  (host Node) — `tools/fonts/regen.sh`, same `evora_*` symbol names so no code churn. Build-validated, fits flash.
- **Palette + squared slabs** across `evora_home/screen/wizard` (macro repoint, `card()` flattened, radius→2).
  Pro surfaces (discipline PRO bar, Pro hub) now **orange**; danger chips **red**; dog recolored **white**.
- **Right-side slide-out app drawer** (`EvoraHome::openDrawer/closeDrawer`, lime pull-tab on every home state):
  `new_model` (lime primary) + `edit_model` + nav (helicopters/system/bind/tools), scrim-to-close, slides via `lv_anim`.
  Home decluttered (inline CTAs + bottom toolbar removed → drawer).
- **Two-tone `EVO`(white)+`RA`(lime) wordmark** ("Evolution Radio") via `lv_label` recolor; identity home
  left-aligned with a **prominent white dog** right (board layout).
- **Flight**: replaced the off-style / mis-rendering **arc gauge with a giant lime headspeed numeral** + flat
  color blocks (Brutalist shows numbers, not dials). **Faint dog watermark added to all sub-pages.**
  NOTE: `lv_img_set_zoom` is a no-op/broken in EdgeTX's lv_conf (transforms off) — watermarks use **native size**.
- Bin: `evora-tx16s-brutalist.bin` (1.75 MB) flashed-ready. Renders in `mockups/gallery/sim-*.png`,
  `board-0{1,6,12}.png`, `board-06x.png`. **TODO:** wizard Brutalist pass; wire MK3 (`PCBREV=TX16SMK3`)
  H7 build target for an 800×480 bin; minor: dog slightly over `FIRMWARE` label edge; status-text truncations on dense tiles.

**2026-06-03 (pm) — Edit Model IA: discipline tiles + chips + per-discipline Pro.**
- Reworked the menu around the owner's VBar-style vision (`Evora-TX@2cfe523f7`). Home now has
  **New model** (wizard) + **Edit model**. Edit Model = a horizontal-scroll grid of **discipline
  tiles** (ESC/Motor, Main Rotor, Swashplate, Tail Rotor, Governor, Flight Tuning, Filters, Receiver,
  Battery, Failsafe, Banks, Blackbox) → a discipline's **essentials as touch chips** + an amber
  **PRO MODE** bar → the discipline's **full settings as chips** (reusing the Configurator-depth
  PTabs), crash-causers red. Data-driven (`renderChips` + `Discipline`), so depth scales. Verified
  across the flow in the 480 sim. Mockups: `design/menu-system.md`, `mockups/evora-edit-*.html`.
- **TODO:** VBar-style right slide-out **drawer** for nav (owner sending a reference shot); New Model
  factory-reset that **preserves RX/bind** (snapshot receiver cfg → reset flight groups → restore);
  per-field **editors** + live values (both wire in with Rotorflight MSP); horizontal **snap-paging**
  (currently free drag); split Main/Tail rotor Pro from the shared mixer PTab.
- **Autonomous follow-up batch (same day):** tap-a-chip **editor** (explain + +/- stepper, local
  overrides until MSP); plain-language **hints on every chip**; **MK3/800 verified** (Edit-Model flow
  scales clean, no defects); **New Model reset design** ([`design/new-model-reset.md`] — snapshot RX →
  reset → restore); **VBar drawer concept** mockup (`mockups/evora-drawer.*`); **System** is now
  device-settings-only (Pro retired from Settings - lives under Edit Model); **Tail Rotor** has its
  own Pro page. Bins through `Evora-TX@cdedbd508`. Remaining: horizontal snap-paging; wire MSP.

**2026-06-03 — menu system + Pro area + ELRS re-pin + visual pass.**
- **ELRS re-pinned `4.0.1 → 3.6.3`** (Evora-Link rebased; ELRS 4.0 breaking changes) — [`upstream-baselines.md`](upstream-baselines.md).
- **Critical visual pass, both resolutions** — fixed the wizard stepper `+/-` glyphs (EVF_NUM_MD is
  numerals-only), the `12 / 112` / `2150 rpm` glyph drops, and the governor numRow label/value overlap.
- **Menu system rebuilt** (`evora_screen.{h,cpp}`, `Evora-TX@b0c04905d`): scrollable list pages + a
  push/pop nav stack; **System** gains a **Pro-mode toggle**; Pro reveals **PRO TUNING — the full
  Rotorflight Configurator surface**. All 18 Pro tabs are **built out** (`Evora-TX@eceaf8f90`) with
  real parameters/ranges/defaults pulled from the Rotorflight firmware `settings.c` — sectioned,
  each setting with a one-line plain-language hint, crash-causers flagged red (data-driven
  PParam/PSection/PTab renderer). Tools gains sub-pages. Design: [`design/menu-system.md`](design/menu-system.md).
  **Values are read-only placeholders until MSP. TODO:** Rotorflight MSP read/write + per-field
  editors (steppers/pickers) to make Pro live; encoder scroll/edit wiring; real `g_eeGeneral` device
  editors; deepen the advanced long-tail params (all servos, PID error-decay/iterm-relax, rescue).

**2026-06-02 — first on-hardware flash + UI fixes + local toolchain.**
- **Local native build solved.** Built a native arm64 toolchain image (ARM aarch64 `arm-none-eabi
  14.2.rel1`) → firmware compiles clean on this Mac with no qemu segfaults. `./build-evora-tx.sh`.
  Also stood up a headless **simulator screenshot loop** (`./sim.sh`) — see [`build-and-simulator.md`](build-and-simulator.md).
- **First flash to the real TX16S** (owner) — booted as Evora; surfaced the backlog below.
- **Fixed + verified in sim, committed (`Evora-TX@e1b561776`):**
  - Wizard **board-orientation** (Flat/Inverted/On side) and **swashplate** (H3-120/H3-140/H1)
    diagrams now **redraw per selection** (were static art). `diagBoard`/`diagSwash` take the index.
  - **"OpenBar" → "EVORA"**: the quick-menu title was the `ICON_TOP_LOGO` **bitmap**
    (`mask_top_logo.png`), never rebranded — regenerated as an EVORA wordmark (both resolutions).
  - **"About EdgeTX" → "About Evora"** (`TR_MAIN_MENU_ABOUT_EDGETX`, en).
- **Boot chime** generated (`dist/hello.wav`, replaces "welcome to EdgeTX" — it's an SD file, not firmware).
- **ELRS pin researched** — recommend `4.0.1 → 3.6.3`, see [`upstream-baselines.md`](upstream-baselines.md).

## Open backlog (from on-hardware testing 2026-06-02)
1. **Scroll wheel adjusts the focused field/trim** (owner request). Real input-layer work: Evora's
   touch-built screens aren't wired to the rotary encoder's LVGL group. **Can't be sim-verified**
   (encoder ≠ touch) — needs hardware iteration.
2. **Ground-up settings, not EdgeTX skins** (owner direction). Model screens mostly *removed* (config
   is baked-in / wizard-driven); **Radio Settings rebuilt ground-up** as native Evora screens (like
   `evora_home`/`evora_wizard`), referencing EdgeTX only for *how the plumbing works*. Needs a design pass.
3. **Branding sweep** — `About EdgeTX` fixed for `en`; other languages (es/nl/fi) + any remaining
   EdgeTX wordmarks/strings still TODO.

## Hardware (owner's bench)
- Radios: RadioMaster **TX16S** (now) + **TX16S MK3** (H7) — both supported. Internal **ELRS** module (`RadioMaster TX16S 2400 TX`).
- FC: **FlyDragon F722 V2** (Rotorflight `FLYDRAGONF722_V2_2`, STM32F722) with **built-in ELRS RX** (ESP8285+SX1280, FC UART1, flash via Rotorflight passthrough; no mainline ELRS target — author one).
- Rotorflight: latest stable **2.2.1** (owner flashes).

## Key docs
- [`build-and-simulator.md`](build-and-simulator.md) — native local build + simulator dev loop (current).
- [`upstream-baselines.md`](upstream-baselines.md) — EdgeTX/ELRS/Rotorflight pins, the ELRS version decision, reference repos.
- `superpowers/specs/2026-06-01-evora-architecture-design.md` — architecture + roadmap.
- `design/radio-is-a-controller.md` — the fixed-channel-map design law (`AECR1T23`).
- `design/heli-setup-parameters.md` — VBar setup params → Rotorflight mapping + the 13-step wizard spec.
- `research/01-04` — Rotorflight / EdgeTX / ELRS / VBar research.
- `emulator/README.md` — emulator harness. `mockups/` — design mockups + wizard. `sdcard/` — per-radio SD assets.
- Memory: `[[evora-project]]`, `[[evora-rc-terminology]]`.
