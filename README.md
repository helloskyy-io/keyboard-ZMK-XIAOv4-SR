# keyboard-ZMK-XIAOv4-SR

ZMK firmware config for a custom split keyboard built with **Seeed Studio XIAO BLE** (nRF52840) and shift-register matrix. This repo is both your **keymap/config** and the source of **custom board + shield** definitions.

---

## Where this comes from

- **Upstream / fork base:** [WainingForests/ZMK-XIAOv4-SR](https://github.com/WainingForests/ZMK-XIAOv4-SR)  
  “Collection of all XIAO-flex V4 with Shift register keyboard firmware.”

- **ZMK:** [zmkfirmware/zmk](https://github.com/zmkfirmware/zmk)  
  The GitHub Action pulls ZMK + Zephyr and builds using **this** repo as the config (and as `ZMK_EXTRA_MODULES` for the custom board and shield).

---

## Version pinning (current setup)

We **pin** ZMK and the trackball driver so builds stay stable and don’t break when upstream changes.

| What | Where | Pinned to |
|------|--------|-----------|
| **ZMK** | `config/west.yml` → `zmk` project | `revision: v0.3` |
| **GitHub workflow** | `.github/workflows/blank.yml` | `@v0.3` (uses ZMK’s v0.3 workflow) |
| **PMW3610 trackball driver** | `config/west.yml` → `zmk-pmw3610-driver` | `revision: zmk-0.3` |

**Why:** Unpinned (`main`) can break when ZMK or Zephyr change (e.g. “No board named 'seeeduino_xiao_ble' found”). Pinning to v0.3 matches the builder’s recommendation and keeps builds working. When you want to upgrade, change the revision/tag in both the workflow and `config/west.yml`.

---

## Repos we use (so we don’t lose them)

| Repo | Role |
|------|------|
| [zmkfirmware/zmk](https://github.com/zmkfirmware/zmk) | Core ZMK firmware (pulled via `config/west.yml`; pinned to v0.3). |
| [WainingForests/ZMK-XIAOv4-SR](https://github.com/WainingForests/ZMK-XIAOv4-SR) | Upstream: board/shield layout (XIAO BLE + shift register). |
| [nickcoutsos/keymap-editor](https://github.com/nickcoutsos/keymap-editor) | Web keymap editor; app: [nickcoutsos.github.io/keymap-editor](https://nickcoutsos.github.io/keymap-editor/). |
| **Your fork** (e.g. [helloskyy-io/keyboard-ZMK-XIAOv4-SR](https://github.com/helloskyy-io/keyboard-ZMK-XIAOv4-SR)) | Your config and keymaps; Keymap Editor points here. |
| [badjeff/zmk-pmw3610-driver](https://github.com/badjeff/zmk-pmw3610-driver) | **Dual trackball (PMW3610)** — listed in `config/west.yml` on layout branches that use trackballs; pinned to `zmk-0.3`. |

---

## Branches: one per keyboard layout

**Use the branch that matches your physical keyboard** (key count, trackballs, OLEDs).

- **main** – one variant (e.g. simpler build).
- **NS-5x6+5-UG-TBx2-OLEDx2** – this keyboard: 5×6+5 keys, underglow, 2 trackballs, 2 OLEDs (`nice_view`). Builds left/right with `nice_view_adapter nice_view`; keymap and overlays live on this branch.

Nothing “trickles down” from main — each branch has its own files and history. The Keymap Editor pushes to **whatever branch you select** in the editor; GitHub Actions then builds that branch.

---

## How the Keymap Editor works

1. Open **[Nick Coutsos’ Keymap Editor](https://nickcoutsos.github.io/keymap-editor/)**.
2. Point it at **your** GitHub repo (e.g. `https://github.com/helloskyy-io/keyboard-ZMK-XIAOv4-SR`).
3. **Choose the branch** that matches your keyboard (e.g. `NS-5x6+5-UG-TBx2-OLEDx2`).
4. The editor loads the latest commit on that branch; you edit the keymap in the UI (layers, combos, behaviors, etc.).
5. When you save, it **commits and pushes** to that branch.
6. **GitHub Actions** runs for that branch and builds the firmware.
7. Download the build artifacts (left/right `.uf2`) from the Actions run and flash each half.

So: **pick branch → edit keymap → save (push) → Actions build → download artifacts → flash.**

---

## How to flash the keyboards

1. **Get the firmware** – After a successful Actions run, open the run and download the **artifacts** (e.g. a zip with left/right `.uf2` or `.bin`).
2. **Left half** – Use the artifact for `skreecustom_left` (e.g. `skreecustom_left-seeduino_xiao_ble-zmk.uf2`).
3. **Right half** – Use the artifact for `skreecustom_right` (e.g. `skreecustom_right-seeduino_xiao_ble-zmk.uf2`).
4. **Enter bootloader** – On the XIAO BLE, double-tap the **RST** (reset) button on the back. The board should appear as a USB drive.
5. **Copy firmware** – Copy the matching `.uf2` file onto that drive. The device will reset and run the new firmware.

Repeat for the other half if you’re flashing both.

---

## Repo layout (what’s what)

| Path | Purpose |
|------|--------|
| `config/` | ZMK config: `west.yml` (pulls ZMK + badjeff PMW3610 driver; revisions pinned), keymap-related config. |
| `build.yaml` | **Build matrix** for GitHub Actions: which board + shield combos to build (left, right, etc.). |
| `boards/` | **Custom board** `seeeduino_xiao_ble` and **shield** `skreecustom` (pinout, matrix, overlays). |
| `zephyr/module.yml` | Tells ZMK/Zephyr to treat this repo as an extra module and use `boards/` as a board root. |
| `.github/workflows/blank.yml` | Calls ZMK’s “build user config” workflow (pinned to `@v0.3`) so each push can build firmware. |

---

## Trackball code locations

| File | Role |
|------|------|
| **`boards/shields/skreecustom/skreecustom.dtsi`** | **Single source of truth** for trackball behavior: base sensitivity, scroll vs pointer, layer-based speed overrides (layer 1 = faster left scroll, layer 2 = faster right pointer). Edit here to change trackball settings. |
| **`skreecustom_left.overlay`** / **`skreecustom_right.overlay`** | Hardware only: enable listeners (`status = "okay"`), connect SPI/device. Do **not** put `input-processors` here. |
| **`skreecustom.keymap`** | Key bindings only. Trackball config is **not** in the keymap; a comment at the bottom points to the `.dtsi`. Do not add `&trackball_*_listener` overrides here (they would replace the dtsi values). |

## If the build fails

- **“No board named 'seeeduino_xiao_ble' found”** – Usually means ZMK/Zephyr moved on and the old flat board layout isn’t supported. **Current fix:** Pin ZMK and the workflow to v0.3 (and the badjeff driver to zmk-0.3) as in this README. If you later switch to ZMK `main`, you may need a different fix (e.g. use board `xiao_ble` and include the overlay from the shield).
- **Check** – Open the failing Actions run and look at the `west build` command and the “Please choose one of the following boards” message to see what’s going on.

---

## Summary

- **Upstream:** [WainingForests/ZMK-XIAOv4-SR](https://github.com/WainingForests/ZMK-XIAOv4-SR). **ZMK:** [zmkfirmware/zmk](https://github.com/zmkfirmware/zmk).
- **Pinning:** ZMK and workflow at **v0.3**; badjeff PMW3610 driver at **zmk-0.3** (see `config/west.yml` and `.github/workflows/blank.yml`).
- **Keymap:** [Keymap Editor](https://nickcoutsos.github.io/keymap-editor/) → pick your repo and branch → edit → push → Actions build.
- **Flash:** Download artifacts from the successful run, put each half in bootloader (double-tap RST), copy the matching `.uf2` onto the drive.
