# Next steps: reference and goals

Definitive setup and decisions. Use for upgrades and future tweaks.

---

## 1. ZMK version (v0.3)

**Current:** ZMK **v0.3** (pinned in `config/west.yml` and `.github/workflows/blank.yml`). `main` is ahead (Zephyr 4.1) but not needed for this board. v0.4 doesn’t exist yet; `main` is ahead with Zephyr 4.1. 
**Plan:** Stay on v0.3 until v0.4 is released and stable, then plan a move.

| ZMK release | Date | Notes |
|-------------|------|--------|
| v0.1.0 | 2024-11-29 | First versioned release |
| v0.2.0 | 2025-03-01 | Toggle mode, mouse/scroll, display config, etc. |
| v0.2.1 | 2025-03-02 | Bug fixes |
| v0.3.0 | 2025-08-01 | **We’re here** – full-duplex wired split, nice!view, pointing/split 
fixes |
| (no v0.4) | — | When v0.4 is released and stable, plan a move. |
**Sleep/idle:** Deep sleep off (`ZMK_SLEEP=n`) and 24 h idle (`ZMK_IDLE_TIMEOUT=86400000`) in **`boards/shields/skreecustom/Kconfig.defconfig`**. That’s the limit of what we can do in this repo without upstream changes.

So we’re **one step “back” from `main`** (which is bleeding edge). v0.3 is **cutting edge 
stable**.

**Useful things in v0.3 (we already have):** Full-duplex wired split, nice!view improvements, 
mouse/scroll, pointing fixes, BLE/profile address by index.

**If we moved to `main` later:** Zephyr 4.1, new boards, Hardware Model v2 – but we’d need 
the “board fix” (e.g. `xiao_ble` + overlay) and possibly badjeff `zmk-0.4` branch.

**Recommendation:** Stay on v0.3 for now (cutting edge, not bleeding edge). When there is a 
**v0.4 tag** and it’s considered a **stable release**, we can plan a move to v0.4 at that 
time.

**Goal:** No sleep or idle effects when keyboard is always on USB.
**Trackballs:** Left = scroll (working well with force-awake). Right = pointer; sensor may be damaged. **Experiment in progress:** 4ms (`force-awake-4ms-mode`) is **commented out on both sides** to test if the right trackball returns; reflash both halves and check.

---

## 2. Sleep and idle (reference)

**Deep sleep vs idle**

| | Idle | Deep sleep |
|---|------|-------------|
| **What** | Low-power after no activity (default 30 s). | Software power-off; only if `ZMK_SLEEP=y`; after idle + `ZMK_IDLE_SLEEP_TIMEOUT` (default 15 min). |
| **What ZMK does** | Turns off peripherals (display, lighting). Stays connected (USB/BLE). | Disconnects (BLE), disables peripherals, can clear RAM. |
| **What you see** | Display/OLED off. Keys still work; no disconnect. | Keyboard disappears; after wake, reconnect (several seconds). |
| **Recovery** | Instant (peripherals re-enable on activity). | ~2–10+ s; first keypresses can be lost. We have deep sleep **off**, so we never enter this. |
| **Trackballs** | When ZMK goes idle, driver **stops** force-awake and sensor downshifts (REST1→REST2→REST3). Wake from REST can take hundreds of ms to several seconds – that was the “~5 s before trackball works” before we set 24 h idle + force-awake. | N/A for us (deep sleep off). |

**Trackballs and sleep:** force-awake keeps the PMW3610 in RUN **only while ZMK is ACTIVE**. When ZMK goes idle/sleep, the driver allows the sensor to downshift. So trackballs **do** sleep when idle kicks in, even with force-awake set. Our 24 h idle keeps ZMK “active” so force-awake stays effective.

- **What it is:** A low-power state ZMK enters after a period of no activity. Default: 30 s 
(`CONFIG_ZMK_IDLE_TIMEOUT`).
- **What ZMK does:** Turns off **peripherals** (displays, lighting, etc.). The keyboard 
**stays connected** (USB or BLE); key presses are still seen and the board is “active” from 
the host’s point of view.
- **What you see:** Display/OLED turns off, underglow/lighting can turn off (if configured). 
Keys still work; first keypress wakes things back up. No disconnect/reconnect.
- **Recovery:** Effectively instant. ZMK is still running and connected; peripherals are 
re-enabled when activity is detected. No “reconnect” step.
- **Trackballs:** When ZMK goes idle, the badjeff driver **stops** force-awake and lets the 
PMW3610 downshift (RUN → REST1 → REST2 → REST3). The sensor then samples less often (40 ms, 
100 ms, 500 ms in deeper REST). When you touch the trackball, the sensor has to **wake from 
REST** and the driver has to bring it back to RUN. That wake-up can take **on the order of 
hundreds of ms to several seconds** (depending how deep into REST it went and driver 
behavior). That matches the “up to ~5 seconds before the trackball starts working” you saw 
**before** we set long idle + force-awake: ZMK was going idle (e.g. after 30 s), the sensor 
went to REST, and waking from REST caused the delay.

**Deep sleep**

- **What it is:** A **software power-off** state. Only entered if `ZMK_SLEEP=y`. After idle, 
ZMK can then enter deep sleep after another timeout (`CONFIG_ZMK_IDLE_SLEEP_TIMEOUT`, default 
15 min).
- **What ZMK does:** Disconnects from **all** connections (BLE); disables peripherals; can 
disable external power; clears RAM (including unsaved Studio state). Board is effectively 
“off” until a wakeup source (e.g. key matrix with `wakeup-source`) triggers.
- **What you see:** BLE keyboard disappears from the host; display is off. After you press a 
key, the board boots from wake-up and has to **reconnect** (BLE pairing/reconnect, USB 
re-enumeration if applicable).
- **Recovery:** **Several seconds** (ZMK docs: “a few seconds to reconnect”). Users often 
report **~2–10+ seconds** depending on BLE stack, host, and conditions. First keypresses 
during wake can be **lost** (keyboard not yet connected). We have **deep sleep disabled** 
(`ZMK_SLEEP=n`), so we never enter this state.

**Going further (no code in this repo):** Idle has no “disabled” in ZMK – only a timeout. Two options if we ever implement: (1) **ZMK** – add “idle off” or timeout=0 → never idle (fixes whole board; bigger change). (2) **badjeff driver** – add “force-awake always” (ignore ZMK state; trackballs only; easier). See [ZMK #2989](https://github.com/zmkfirmware/zmk/issues/2989) for BLE-related sleep requests.

---

## 3. Trackballs

**Where code lives:** See **README → Trackball code locations**. Summary:

- **`boards/shields/skreecustom/skreecustom.dtsi`** – Single source of truth: base sensitivity, scroll vs pointer, **layer-based speed** (layer 1 = 2× left scroll, layer 2 = 2× right pointer). Edit here.
- **`skreecustom_left.overlay`** / **`skreecustom_right.overlay`** – Hardware only (enable listeners, SPI, device). No `input-processors`.
- **`skreecustom.keymap`** – Key bindings only. No trackball overrides (comment in file explains why).

**Current setup:** Left = scroll, right = pointer. **force-awake** and **force-awake-4ms-mode** on both. Base speed is slower for high accuracy; layer-controlled speed is 100% higher than base (2×) for fast movements—e.g. traversing the whole screen or scrolling a long distance—when you hold the layer toggle (mo 1 left, mo 2 right). Both trackballs working well.

**Lesson (hardware):** IPA can distort resin trackball mounts and change ball–sensor spacing. Use dry brush, compressed air, or resin-safe cleaner on trackball parts.

**badjeff driver:** Rotation = swap/invert only (no degrees). Optional: open an issue on badjeff/zmk-pmw3610-driver for rotation in degrees and link it here.

---

## 4. Plan summary

| Goal | Status |
|------|--------|
| **ZMK v0.3 → v0.4** | Stay on v0.3; move when v0.4 is stable. |
| **Sleep / idle** | Done: `ZMK_SLEEP=n`, `ZMK_IDLE_TIMEOUT=24h` in Kconfig.defconfig. |
| **Trackballs** | Done: config in dtsi, layer speed in dtsi, both sides working. Avoid IPA on resin parts. |
| **Rotation** | Use swap/invert; optional badjeff issue for rotation in degrees. |
| **badjeff v0.4** | When upgrading ZMK to v0.4, switch driver to zmk-0.4 and any “alt” names. |

---

## 5. Future ideas (optional)

- **Precision vs speed layers:** Base layer at even lower speed for added sensitivity, another layer at 3× (we are currently on 2× for layered control).
- **Swap roles per layer:** e.g. left = pointer / right = scroll on a different layer (mouse on left).
- **Scroll axis per layer:** Horizontal-only scroll or swap scroll axes via scroll transform processors. (Future add for 3D modeling potentially.)

No urgency; the keyboard is in a great state. Revisit when you want to experiment.
