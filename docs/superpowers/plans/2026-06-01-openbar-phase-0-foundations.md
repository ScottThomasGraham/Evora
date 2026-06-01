# OpenBar Phase 0 — Foundations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove we can fork, build, flash, and recover *stock* versions of all three OpenBar firmwares on the maintainer's actual hardware — establishing the toolchains, repos, and recovery procedures before we change any behavior.

**Architecture:** OpenBar is three forks (radio = EdgeTX, link TX + RX = ExpressLRS) riding an untouched Rotorflight FC. Phase 0 does **no feature work** — it de-risks "can we own each end?" The riskiest item (flashing our own ELRS build onto the FlyDragon's built-in, vendor-targeted RX) is proven here.

**Tech Stack:** EdgeTX (C++/CMake, Docker build), ExpressLRS (C++/PlatformIO), Rotorflight (flashed by the maintainer, not built by us), GitHub forks + Actions CI.

**Roles:** Steps are tagged **[CLAUDE]** (Claude runs on the Mac) or **[USER]** (maintainer physically flashes hardware and reports observations — Claude cannot touch the radio/heli). Build = Claude; flash + bench-verify = user.

**Reference hardware (confirmed 2026-06-01):**
- Radio: RadioMaster **TX16S**, STM32F407, internal **ELRS** module → ELRS TX target `RadioMaster TX16S 2400 TX`.
- FC: **FlyDragon F722 V2 / V2.2** FBL, STM32F722, Rotorflight target `FLYDRAGONF722_V2_2` (confirm exact revision), **latest stable Rotorflight 2.2.1** (maintainer flashes this themselves).
- RX: **built-in ELRS** on the FlyDragon (ESP8285 + SX1280, 2.4 GHz, on F722 UART1, CRSF), flashed via Rotorflight serial passthrough. **No mainline ELRS target exists — we author one.**

**Workspace layout:** upstream forks are cloned under `~/Projects/OpenBar/forks/` (gitignored; they are their own GitHub repos, not vendored into the OpenBar meta repo).

---

## File / repo structure (decided here)

| Repo | Origin | Purpose |
|---|---|---|
| `ScottThomasGraham/OpenBar` (exists) | — | Meta repo: docs, specs, plans, build orchestration, confirmed-config records. |
| `ScottThomasGraham/OpenBar-Radio` | fork of `edgetx/edgetx` | OpenBar radio firmware. Work on branch `openbar`. |
| `ScottThomasGraham/OpenBar-Link` | fork of `ExpressLRS/ExpressLRS` | OpenBar link firmware (TX module + RX). Work on branch `openbar`. |

Local clones: `~/Projects/OpenBar/forks/OpenBar-Radio`, `~/Projects/OpenBar/forks/OpenBar-Link`.

Phase 0 also creates, in the **meta repo**:
- `docs/hardware/confirmed-config.md` — the single source of truth for exact targets, versions, board revision, binding phrase location, recovery steps.
- `forks/.local/binding.env` — **untracked** (gitignored) — the chosen ELRS binding phrase.

---

## Task 0: Verify the build environment [CLAUDE]

**Files:** none (environment only).

- [ ] **Step 1: Check Docker (needed for EdgeTX builds)**

Run: `docker --version && docker info >/dev/null 2>&1 && echo "docker ok"`
Expected: a version string and `docker ok`. If Docker is missing or the daemon isn't running, stop and tell the user to install/start **Docker Desktop** before continuing.

- [ ] **Step 2: Check Python + PlatformIO (needed for ExpressLRS builds)**

Run: `python3 --version && (pio --version 2>/dev/null || echo "pio missing")`
Expected: Python ≥ 3.9. If `pio missing`, install it:
Run: `python3 -m pip install --user platformio && pio --version`
Expected: a PlatformIO Core version string.

- [ ] **Step 3: Check git + gh CLI auth (needed to create forks)**

Run: `git --version && gh auth status 2>&1 | head -5`
Expected: gh reports logged in as `ScottThomasGraham`. If not logged in, the user must run `! gh auth login` (interactive) in the session, or we fall back to forking via the GitHub web UI in Task 1.

- [ ] **Step 4: Create the forks workspace and gitignore guard**

Run: `mkdir -p ~/Projects/OpenBar/forks/.local && echo "forks workspace ready"`
Expected: `forks workspace ready`. (`/forks/` is already in `.gitignore`.)

---

## Task 1: Fork and clone the firmwares [CLAUDE] (+ [USER] if gh not authed)

**Files:**
- Create: `~/Projects/OpenBar/forks/OpenBar-Radio/` (clone)
- Create: `~/Projects/OpenBar/forks/OpenBar-Link/` (clone)
- Create: `~/Projects/OpenBar/forks/OpenBar-Radio/OPENBAR.md`, `.../OpenBar-Link/OPENBAR.md`

- [ ] **Step 1: Fork EdgeTX → OpenBar-Radio**

Run: `gh repo fork edgetx/edgetx --fork-name OpenBar-Radio --clone=false --org ScottThomasGraham 2>&1 || gh repo fork edgetx/edgetx --fork-name OpenBar-Radio --clone=false`
Expected: a fork `ScottThomasGraham/OpenBar-Radio` is created. (If gh is unauthed: **[USER]** clicks "Fork" on https://github.com/edgetx/edgetx, renames to `OpenBar-Radio`, then tell Claude it's done.)

- [ ] **Step 2: Fork ExpressLRS → OpenBar-Link**

Run: `gh repo fork ExpressLRS/ExpressLRS --fork-name OpenBar-Link --clone=false 2>&1`
Expected: a fork `ScottThomasGraham/OpenBar-Link` is created. (Web-UI fallback as above if needed.)

- [ ] **Step 3: Clone both forks (shallow-ish, with submodules for EdgeTX)**

Run:
```bash
cd ~/Projects/OpenBar/forks
git clone --recurse-submodules https://github.com/ScottThomasGraham/OpenBar-Radio.git
git clone --recurse-submodules https://github.com/ScottThomasGraham/OpenBar-Link.git
```
Expected: both repos clone, including submodules (EdgeTX and ELRS both use submodules). Verify: `ls OpenBar-Radio/radio && ls OpenBar-Link/src`.

- [ ] **Step 4: Add upstream remotes and record the pinned baseline**

Run:
```bash
cd ~/Projects/OpenBar/forks/OpenBar-Radio && git remote add upstream https://github.com/edgetx/edgetx.git && git fetch --tags upstream && git rev-parse HEAD
cd ~/Projects/OpenBar/forks/OpenBar-Link && git remote add upstream https://github.com/ExpressLRS/ExpressLRS.git && git fetch --tags upstream && git rev-parse HEAD
```
Expected: two commit SHAs printed. Note: prefer pinning to the latest **stable release tag** of each (EdgeTX: latest `v2.x.x`; ELRS: latest `3.x.x`). Check out those tags onto an `openbar` branch in Step 5.

- [ ] **Step 5: Create the `openbar` working branch at the latest stable tag**

Run (replace tags with the actual latest stable shown by `git tag --sort=-creatordate | head`):
```bash
cd ~/Projects/OpenBar/forks/OpenBar-Radio && git tag --sort=-creatordate | grep -E '^v?2\.' | head -5
cd ~/Projects/OpenBar/forks/OpenBar-Link && git tag --sort=-creatordate | grep -E '^v?3\.' | head -5
```
Then, for each repo: `git checkout -b openbar <latest-stable-tag> && git submodule update --init --recursive`
Expected: each repo is on branch `openbar` at a tagged stable commit, submodules synced.

- [ ] **Step 6: Write OPENBAR.md (maintenance strategy) in each fork**

Create `OPENBAR.md` in each fork with this content (adjust SHAs/tags to the recorded values):
```markdown
# OpenBar fork

Upstream: <edgetx/edgetx | ExpressLRS/ExpressLRS>
Pinned baseline: <tag> (<SHA>)
Working branch: openbar

## Maintenance
- Keep OpenBar changes minimal-diff and layered so upstream rebases stay tractable.
- To update: `git fetch upstream --tags`, branch from new stable tag, cherry-pick/rebase the openbar diff, rebuild + re-run Phase 0 flash/recovery drills before adopting.
- Never modify Rotorflight; never touch flight-safety plumbing (failsafe/arming/ADC) unless a task explicitly requires it.
```

- [ ] **Step 7: Commit and push the branch**

Run (each fork):
```bash
git add OPENBAR.md && git commit -m "chore: add OpenBar fork maintenance notes" && git push -u origin openbar
```
Expected: branch `openbar` pushed to each fork.

---

## Task 2: Build stock EdgeTX for the TX16S [CLAUDE]

**Files:**
- Create: `~/Projects/OpenBar/forks/OpenBar-Radio/build-tx16s/` (build dir)

- [ ] **Step 1: Pull the EdgeTX dev build container**

Run: `docker pull ghcr.io/edgetx/edgetx-dev`
Expected: image pulled. (This is EdgeTX's official build environment; it contains the ARM toolchain + deps.)

- [ ] **Step 2: Configure + build the TX16S firmware in the container**

Run:
```bash
cd ~/Projects/OpenBar/forks/OpenBar-Radio
docker run --rm -v "$PWD:/src" ghcr.io/edgetx/edgetx-dev bash -lc '
  cd /src && mkdir -p build-tx16s && cd build-tx16s &&
  cmake -DPCB=X10 -DPCBREV=TX16S -DDEFAULT_MODE=2 -DCMAKE_BUILD_TYPE=Release ../ &&
  make -j"$(nproc)" firmware'
```
Expected: build completes with `firmware.bin` produced. (Cross-check flags against `OpenBar-Radio/doc/` / the EdgeTX "Compiling firmware" docs if cmake errors; `PCB=X10 PCBREV=TX16S` is the TX16S family.)

- [ ] **Step 3: Verify the artifact**

Run: `ls -la ~/Projects/OpenBar/forks/OpenBar-Radio/build-tx16s/firmware.bin`
Expected: a `firmware.bin` of roughly 2–4 MB. Copy it to a clear name:
Run: `cp ~/Projects/OpenBar/forks/OpenBar-Radio/build-tx16s/firmware.bin ~/Projects/OpenBar/forks/.local/edgetx-tx16s-stock.bin`

---

## Task 3: Flash stock EdgeTX to the TX16S + recovery drill [USER]

**Files:** none (hardware).

- [ ] **Step 1: Back up the current radio firmware/state**

[USER] On the radio: copy the existing SD card contents to the Mac as a backup, and note the current EdgeTX version (Radio Settings → Version). This is the known-good fallback.

- [ ] **Step 2: Flash the freshly built firmware**

[USER] Put the radio in **bootloader/DFU mode** (hold both horizontal trims inward while powering on → "DFU" / USB storage). Connect USB. Copy `forks/.local/edgetx-tx16s-stock.bin` onto the radio's FIRMWARE volume (or flash via EdgeTX Companion / `Write Firmware`). Power-cycle.

- [ ] **Step 3: Verify it boots and sticks work**

[USER] Confirm the radio boots EdgeTX, version matches the built stable tag, and gimbals/switches respond in the Channel Monitor. Report result to Claude.
Acceptance: boots cleanly + inputs read correctly.

- [ ] **Step 4: Recovery drill**

[USER] Re-enter bootloader mode and re-flash to prove recovery works (flash the backup, or re-flash the same bin). Confirm the radio survives a re-flash. Report result.
Acceptance: a bad/again flash is always recoverable via bootloader. **Document the exact recovery steps that worked** for `confirmed-config.md`.

---

## Task 4: Build stock ELRS for the TX16S internal module [CLAUDE]

**Files:**
- Modify (transiently): ELRS build options for target `RadioMaster TX16S 2400 TX`.

- [ ] **Step 1: [USER] Confirm the exact internal-module target + current version**

[USER] On the radio: EdgeTX → Tools → run the **ExpressLRS** Lua script. Read the header (maker/model/firmware version). Confirm it identifies as a **RadioMaster 2.4 GHz internal** module (target family `RadioMaster TX16S 2400 TX`). Report the exact model string + version to Claude. Record in `confirmed-config.md`.

- [ ] **Step 2: Choose and store the OpenBar binding phrase**

[USER] Pick a binding phrase (any memorable string, e.g. `openbar-<word>`). [CLAUDE] store it untracked:
Run: `printf 'BINDING_PHRASE=%s\n' "<phrase>" > ~/Projects/OpenBar/forks/.local/binding.env && echo stored`
Expected: `stored`. (This file is gitignored; the binding phrase is link-identifying, keep it out of the public repo.)

- [ ] **Step 3: List available ELRS build environments and find the TX16S internal env**

Run:
```bash
cd ~/Projects/OpenBar/forks/OpenBar-Link
grep -ril "TX16S" targets/ | head; grep -rl "2400" targets/*TX* 2>/dev/null | head
```
Expected: locate the target JSON for the RadioMaster TX16S 2.4 GHz internal module. Record the exact PlatformIO env / target id (`$ELRS_TX_ENV`).

- [ ] **Step 4: Build the TX module firmware with the binding phrase**

Run:
```bash
cd ~/Projects/OpenBar/forks/OpenBar-Link
source ~/Projects/OpenBar/forks/.local/binding.env
pio run -e "$ELRS_TX_ENV"
```
Expected: build succeeds, a `firmware.bin` is produced under `.pio/build/$ELRS_TX_ENV/`. (Binding phrase is applied via the ELRS options/target system; if the env needs the phrase injected differently, use the `options.json`/`-D MY_BINDING_PHRASE` mechanism documented in the ELRS source.)

- [ ] **Step 5: Verify the artifact**

Run: `ls -la ~/Projects/OpenBar/forks/OpenBar-Link/.pio/build/$ELRS_TX_ENV/firmware.bin && cp $_ ~/Projects/OpenBar/forks/.local/elrs-tx16s-stock.bin`
Expected: a firmware binary exists and is copied to `.local/`.

---

## Task 5: Flash stock ELRS to the TX16S internal module + bind + recovery [USER]

**Files:** none (hardware).

- [ ] **Step 1: Flash the internal module**

[USER] Flash the built TX firmware to the internal module using the **ExpressLRS Configurator** ("Local" / build-from-folder pointing at the fork) OR via the on-radio WiFi/UART method for the internal module. (Internal modules typically flash over WiFi or via the radio's module UART — use whichever the ELRS Lua/Configurator offers for this module.) Report the flashed version.

- [ ] **Step 2: Verify the module runs + telemetry to radio**

[USER] Re-open the ELRS Lua script; confirm it shows the freshly built version and the module is alive. Report.
Acceptance: module boots our build and is reachable.

- [ ] **Step 3: Recovery drill**

[USER] Re-flash the module (same bin) to confirm recovery; note the working procedure. If WiFi flashing fails, confirm the UART/passthrough fallback. Document in `confirmed-config.md`.
Acceptance: module is re-flashable/recoverable.

(Binding to the RX happens in Task 7, once the RX runs our firmware with the same phrase.)

---

## Task 6: Build stock ELRS for the FlyDragon built-in RX — author the custom target [CLAUDE] ⚠ key de-risk

**Files:**
- Create: `~/Projects/OpenBar/forks/OpenBar-Link/targets/openbar-flydragon-r24d.json` (custom hardware definition)

- [ ] **Step 1: Gather the RX pin/hardware mapping**

[CLAUDE] The RX is ESP8285 + SX1280, 2.4 GHz diversity. There is no mainline target. Gather the GPIO/pin layout from: FlyDragon's published ELRS receiver manual/firmware, any community hardware.json for "FD R24D" / FlyDragon, and the closest reference target in `targets/` (a generic ESP8285 2400 RX with diversity). Record candidate pin assignments (RF busy/reset/DIO, antenna switch, LED, boot button, UART) in the task notes.

- [ ] **Step 2: Author the custom hardware definition**

Create `targets/openbar-flydragon-r24d.json` modeled on an existing ESP8285 2400 diversity RX target in `targets/`, with the gathered pin map. Keep it minimal and clearly commented as OpenBar-authored.

- [ ] **Step 3: Build the RX firmware against the custom target**

Run:
```bash
cd ~/Projects/OpenBar/forks/OpenBar-Link
source ~/Projects/OpenBar/forks/.local/binding.env
pio run -e <esp8285 RX env using openbar-flydragon-r24d>   # record as $ELRS_RX_ENV
```
Expected: a build completes producing an RX `firmware.bin`. If pin assumptions are wrong the build still compiles (pins are data) — correctness is verified on hardware in Task 7. If the custom target proves intractable, **fallback:** build the closest **Generic ESP8285 2400 RX** target and note the limitation.

- [ ] **Step 4: Verify + stage the artifact**

Run: `ls -la ~/Projects/OpenBar/forks/OpenBar-Link/.pio/build/$ELRS_RX_ENV/firmware.bin && cp $_ ~/Projects/OpenBar/forks/.local/elrs-flydragon-rx.bin`
Expected: RX firmware staged in `.local/`.

---

## Task 7: Flash custom ELRS to the built-in RX via passthrough + verify + recovery [USER] ⚠ key de-risk

**Files:** none (hardware).

- [ ] **Step 1: BACK UP the factory RX firmware first**

[USER] **Critical:** download/keep FlyDragon's factory RX firmware (`FlyDragon R24D ...bin`) as the known-good fallback before flashing anything custom. Confirm you have it saved.

- [ ] **Step 2: Enter RX flash mode via Rotorflight passthrough**

[USER] Connect the FC (FlyDragon F722) to the Mac over USB. In the Rotorflight Configurator CLI (or via `serialpassthrough`), put UART1 into passthrough: the FlyDragon manual specifies `serialpassthrough 0 0 rxtx none` (UART1 = serial index 0). Hold the RX **boot button** until its LED is solid. Report ready.

- [ ] **Step 3: Flash our RX build**

[USER] Using the ELRS Configurator (UART/passthrough flash) or `flash_download_tool` (chip = ESP8285), flash `forks/.local/elrs-flydragon-rx.bin`. Report success/failure and any error.

- [ ] **Step 4: Verify bind + RC control + telemetry**

[USER] Power TX (internal module, same binding phrase) and the heli (props off / motor disabled). Confirm: the RX **binds** automatically (phrase match), Rotorflight shows **RC channels moving** with stick input (Receiver tab), and **link telemetry** (RSSI/LQ) appears on the radio. Report each.
Acceptance: **we built and flashed our own ELRS firmware onto the built-in RX, and it binds + controls + reports telemetry.** This proves OpenBar can own the RX end. ✅

- [ ] **Step 5: Recovery drill (and decision record)**

[USER] Re-flash the **factory** firmware via the same passthrough method to confirm full recovery, then re-flash ours. Confirm both directions work. [CLAUDE] record the outcome in `confirmed-config.md`:
- If custom target worked → record exact pin map + env as canonical.
- If only the generic target worked → record the limitation + follow-up.
- If neither flashed → **architecture flag:** record that the built-in RX is not yet ownable; Phase 1 must either solve the target or plan around an external ELRS RX. (This is the whole point of proving it now.)

---

## Task 8: Continuous Integration [CLAUDE]

**Files:**
- Create: `~/Projects/OpenBar/forks/OpenBar-Radio/.github/workflows/openbar-build.yml`
- Create: `~/Projects/OpenBar/forks/OpenBar-Link/.github/workflows/openbar-build.yml`

- [ ] **Step 1: Add a radio CI workflow that builds the TX16S target**

Create `OpenBar-Radio/.github/workflows/openbar-build.yml`:
```yaml
name: OpenBar Radio build
on: { push: { branches: [openbar] }, pull_request: { branches: [openbar] } }
jobs:
  tx16s:
    runs-on: ubuntu-latest
    container: ghcr.io/edgetx/edgetx-dev
    steps:
      - uses: actions/checkout@v4
        with: { submodules: recursive }
      - run: mkdir build && cd build && cmake -DPCB=X10 -DPCBREV=TX16S -DDEFAULT_MODE=2 ../ && make -j2 firmware
      - uses: actions/upload-artifact@v4
        with: { name: edgetx-tx16s, path: build/firmware.bin }
```

- [ ] **Step 2: Add a link CI workflow that builds the TX + RX targets**

Create `OpenBar-Link/.github/workflows/openbar-build.yml`:
```yaml
name: OpenBar Link build
on: { push: { branches: [openbar] }, pull_request: { branches: [openbar] } }
jobs:
  firmware:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { submodules: recursive }
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install platformio
      - run: pio run -e "$ELRS_TX_ENV"   # set to the recorded TX env
      - run: pio run -e "$ELRS_RX_ENV"   # set to the recorded RX env
```

- [ ] **Step 3: Push and confirm green**

Run (each fork): `git add .github/workflows/openbar-build.yml && git commit -m "ci: build OpenBar targets" && git push`
Then: `gh run watch` (or check the Actions tab).
Expected: both workflows pass. Fix env names/flags until green.

---

## Task 9: Phase 0 wrap-up [CLAUDE]

**Files:**
- Create: `~/Projects/OpenBar/docs/hardware/confirmed-config.md`
- Modify: `~/Projects/OpenBar/README.md` (Status line)

- [ ] **Step 1: Write the confirmed-config record**

Create `docs/hardware/confirmed-config.md` capturing: exact EdgeTX stable tag built, exact ELRS stable tag, TX module target string, RX custom-target outcome + pin map, FlyDragon board revision + Rotorflight target + version flashed, binding-phrase file location, and the **verified recovery procedure for each device**.

- [ ] **Step 2: Update repo status**

Edit `README.md` Status line to: `Phase 0 complete — all three firmwares fork/build/flash/recover verified on hardware. Phase 1 (private fast-config channel) next.` (Adjust wording to actual outcome, especially the RX-target result.)

- [ ] **Step 3: Commit the meta repo**

Run:
```bash
cd ~/Projects/OpenBar && git add -A && git commit -m "docs: Phase 0 foundations complete — confirmed hardware config + recovery"
```
Expected: committed. (Push when the user asks.)

---

## Phase 0 acceptance criteria

Phase 0 is done when **all** are true (or the RX item has a documented decision):
1. Both forks exist, clone, and build their stock target locally **and in CI**.
2. The TX16S boots our EdgeTX build and is recoverable via bootloader.
3. The TX16S internal ELRS module runs our ELRS build and is recoverable.
4. **The FlyDragon built-in RX runs *our* ELRS build, binds, controls, and reports telemetry — and is recoverable to factory** (or: a documented decision if the custom target couldn't be made to work).
5. `docs/hardware/confirmed-config.md` records every exact target/version and the recovery steps.

## Self-review notes

- **Spec coverage:** Implements spec §5 "Phase 0 — Foundations & toolchains" (fork, build, flash, recovery, rebase strategy, CI) and resolves spec §8 open questions 1 (ELRS hardware), 3 (RX/heli), 4 (hosting); open question 2 (Rotorflight version) resolved to 2.2.1.
- **Known runtime-discovered values** (not placeholders): `$ELRS_TX_ENV`, `$ELRS_RX_ENV`, exact stable tags, RX pin map, binding phrase — each has an explicit step that produces and records it before later steps consume it.
- **Risk owned:** Tasks 6–7 isolate the one architecture risk (vendor-targeted built-in RX) and force an early go/no-go with a defined fallback.
