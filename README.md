# keyboard-ZMK-XIAOv4-SR

ZMK firmware config for a custom split keyboard built with **Seeed Studio XIAO BLE** (nRF52840) and shift-register matrix. This repo is both your **keymap/config** and the source of **custom board + shield** definitions, so the layout is not a “standard” ZMK config-only repo.

---

## Where this comes from

- **Upstream / fork base:** [WainingForests/ZMK-XIAOv4-SR](https://github.com/WainingForests/ZMK-XIAOv4-SR)  
  “Collection of all XIAO-flex V4 with Shift register keyboard firmware.”

- **ZMK:** [zmkfirmware/zmk](https://github.com/zmkfirmware/zmk)  
  This repo uses ZMK’s “user config” build: the GitHub Action calls ZMK’s workflow, which pulls ZMK + Zephyr and builds using **this** repo as the config (and as `ZMK_EXTRA_MODULES` for the custom board and shield).

---

## Branches: one per keyboard layout

**Yes — you’re supposed to work in the branch that matches your specific keyboard** (what it looks like: key count, trackballs, OLEDs, etc.).

- **Each branch is a full, self-contained config** for one keyboard variant. For example:
  - **main** – one variant (e.g. simpler build: no display in the matrix).
  - **NS-5x6+5-UG-TBx2-OLEDx2** – this specific keyboard: 5×6+5 keys, underglow, 2 trackballs, 2 OLEDs (`nice_view`). Its `build.yaml` builds left/right with `nice_view_adapter nice_view`; its keymap and overlays live in that branch.
- **Nothing “trickles down” from main.** Git doesn’t work that way — each branch has its own history and files. If you want a fix (e.g. board name, README) on a layout branch, you merge main into that branch (or re-apply the change there).
- **Keymap Editor** – you pick the branch (e.g. `NS-5x6+5-UG-TBx2-OLEDx2`) in the editor; when you save, it pushes to **that** branch. GitHub Actions then runs for that branch and builds from that branch’s `build.yaml` and config.
- **Summary:** Use the branch that describes your physical keyboard; do keymap edits and builds from that branch. Treat main as another layout (or a place to keep shared fixes you then merge into layout branches).

---

## Editing keymaps and flashing

### Keymap Editor (recommended for keymap changes)

1. Open **[Nick Coutsos’ Keymap Editor](https://nickcoutsos.github.io/keymap-editor/)**.
2. Point it at **your** GitHub repo (e.g. `https://github.com/<your-username>/keyboard-ZMK-XIAOv4-SR`).
3. It uses the **latest commit** on the branch you choose, lets you edit the keymap in the UI, then **commits and pushes** to that repo.
4. After the push, **GitHub Actions** in this repo run and build the firmware.
5. Download the built artifacts (e.g. `.uf2` or `.bin`) from the Actions run and flash each half.

So: **last successful commit → keymap editor → push → Actions build → flash the two halves.**

### Flashing the two halves

- **Left:** use the build artifact for `skreecustom_left` (e.g. `skreecustom_left-seeduino_xiao_ble-zmk.uf2`).
- **Right:** use the build artifact for `skreecustom_right` (e.g. `skreecustom_right-seeduino_xiao_ble-zmk.uf2`).

Put the MCU in bootloader mode (e.g. double-tap reset on XIAO BLE), then copy the matching `.uf2` onto the exposed drive.

---

## Repo layout (what’s what)

| Path | Purpose |
|------|--------|
| `config/` | ZMK config: `west.yml` (pulls ZMK), keymap-related config. |
| `build.yaml` | **Build matrix** for GitHub Actions: which board + shield combos to build (left, right, settings_reset, etc.). |
| `boards/` | **Custom board** `seeeduino_xiao_ble` and **shield** `skreecustom` (pinout, matrix, overlays). |
| `zephyr/module.yml` | Tells ZMK/Zephyr to treat this repo as an extra module and use `boards/` as a board root. |
| `.github/workflows/` | Calls ZMK’s “build user config” workflow so each push can build firmware. |

The build uses **this repo** as `ZMK_EXTRA_MODULES` so that the `seeeduino_xiao_ble` board and `skreecustom` shield are found.

---

## If the GitHub Action build fails

Typical failure: **“No board named 'seeeduino_xiao_ble' found.”**

- The workflow **does** pass this repo as `ZMK_EXTRA_MODULES`; the board must be defined in the **Zephyr v2** style under `boards/` (e.g. `boards/arm/seeeduino_xiao_ble/` with a `board.cmake`), not only as a flat `.conf`/`.overlay` at the top of `boards/`.
- If you’re on an older layout and ZMK/Zephyr have been updated, you may need to add that proper board directory (see “Build fix” below) or pin ZMK to a known-good revision in `config/west.yml`.

Checking the **Actions log** for the exact `west build` command and the “Please choose one of the following boards” list will confirm whether the board is being found.

---

## Build fix (board discovery) — applied

The error **"No board named 'seeeduino_xiao_ble' found"** happens because Zephyr 4.x only discovers boards that follow the proper structure (e.g. `boards/<arch>/<board_name>/` with `board.cmake`). This repo only had flat files (`boards/seeeduino_xiao_ble.conf` and `boards/seeeduino_xiao_ble.overlay`), which are no longer enough.

**Fix applied in this repo:**

1. **Use the base board `xiao_ble`** (Seeed XIAO BLE in Zephyr) in `build.yaml` instead of `seeeduino_xiao_ble`.
2. **Apply the custom SPI/WS2812 overlay from the shield:**  
   `boards/shields/skreecustom/skreecustom_left.overlay` and `skreecustom_right.overlay` now `#include "../../seeeduino_xiao_ble.overlay"`.
3. **`CONFIG_SPI=y`** is already set in `boards/shields/skreecustom/Kconfig.defconfig`.

So the firmware is still built for the same hardware; only the board name and where the overlay is applied have changed. The keymap editor → push → Actions flow should work again. The old `boards/seeeduino_xiao_ble.conf` and `boards/seeeduino_xiao_ble.overlay` are kept for reference; the overlay is used via the shield includes.

---

## Summary

- **Upstream:** [WainingForests/ZMK-XIAOv4-SR](https://github.com/WainingForests/ZMK-XIAOv4-SR).
- **Keymap edits:** [Keymap Editor](https://nickcoutsos.github.io/keymap-editor/) → your repo → push → Actions build.
- **Firmware:** Download from Actions artifacts and flash left/right `.uf2` to each XIAO BLE half.
