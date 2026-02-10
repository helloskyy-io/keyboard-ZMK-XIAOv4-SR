# Next steps: research and goals

What we learned and concrete next steps. Use it to decide priorities and implement.

---

## 1. ZMK version (v0.3)

**ZMK:** We’re on **v0.3** (latest stable). v0.4 doesn’t exist yet; `main` is ahead with Zephyr 4.1. **Plan:** Stay on v0.3 until v0.4 is released and stable, then plan a move.

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

**Where code lives:** Per-side overlays – left = **`skreecustom_left.overlay`** (node `trackball_central`), right = **`skreecustom_right.overlay`** (node `trackball_peripheral`). Shared **`skreecustom.dtsi`** defines listeners and the scroll/pointer pipeline. Each half’s overlay defines the sensor on that half; no single file “applies to both.”

**Current setup**

- **Left** = scroll. **Right** = pointer. **Experiment #2 in progress:** force-awake **commented out on both sides** (4ms already commented out). Experiment #1 (4ms off both) did **not** bring back the right trackball – so 4ms was **not** the cause; we can use 4ms without issues. Test tonight: if right returns “halfway” (jumping around) = force-awake on both was causing right to totally stop; if right still dead = sensor likely died; also tests whether force-awake is needed for sensor sleep (left behavior after 10–30 s idle).

**force-awake-4ms-mode:** When enabled, sampling is 4 ms (250 Hz) instead of 8 ms (125 Hz). Better tracking on USB. No documented “4ms + two trackballs” limitation; each half has one trackball and its own firmware.

**Layers:** Trackballs are **not** standalone. They go through **input listeners**; listeners use **input-processors** (pointer vs scroll, sensitivity). In the **keymap** you can override a listener’s input-processors, and ZMK supports **layer-specific** overrides – so you can change trackball behavior per layer (e.g. right = pointer on layer 0, scroll on layer 3). Our keymap has commented-out overrides for both listeners; they can be made layer-conditional. [Input Processor Usage](https://zmk.dev/docs/keymaps/input-processors/usage).

**Pointer jumping (when it happened):** Either **timing** (left/scroll sending REL_X/REL_Y before scroll mapper converts; burst leaks to pointer) or **shared path on central** (both scroll and pointer streams aggregated on central; ordering/race could cause jump). Fix would likely be in the driver and/or ZMK split/pointing. **Biggest issue = dual trackball driver + ZMK split pointing**, not ZMK core. You don’t have to switch firmware; improving the driver or split/pointing can address it.

---

## 4. badjeff and rotation

- **zmk-0.4:** Exists for future ZMK 0.4 (alt compatible/config). Stay on zmk-0.3 until we move to v0.4; then switch driver to zmk-0.4 and “alt” names.
- **Rotation:** badjeff offers **swap-xy**, **invert-x**, **invert-y** only → eight orientations (90° steps). No rotation in degrees. Use swap/invert for now; **submit an issue in badjeff** (zmk-pmw3610-driver) for rotation in degrees; link it here once created.

---

## 5. Plan summary

| Goal | Status / next step |
|------|---------------------|
| **ZMK v3 → v4** | Stay on v0.3; move when v0.4 is stable. |
| **Sleep / idle** | Done: ZMK_SLEEP=n, ZMK_IDLE_TIMEOUT=24h in Kconfig.defconfig. |
| **Trackballs** | **Experiment #2:** force-awake **commented out on both** (4ms already off). Exp #1: 4ms off did NOT bring back right → 4ms not the cause, safe to use. Test: does right return halfway? need force-awake for sensor sleep? |
| **Rotation** | Use swap/invert. Submit badjeff issue for rotation in degrees; link here. |
| **badjeff v0.4** | Stay on zmk-0.3; when upgrading ZMK to v0.4, switch driver to zmk-0.4 and “alt” names. |

---

## 6. Implementation / experiments

**1. Do we need force-awake when ZMK idle/sleep are disabled?**  
Remove force-awake from both sides (keep 24 h idle, deep sleep off). Don’t touch the trackballs for 10–30 s, then use them. If there’s delay again, force-awake was still doing something (e.g. blocking the sensor’s own motion-based RUN→REST downshift). If no delay, we might not need force-awake when ZMK never goes idle/sleep. **Experiment #2 (in progress):** force-awake commented out on both; test tonight – right returns halfway? sensor sleep on left?

**2. Experiment #1 result: 4ms was NOT the cause.**  
Removing 4ms from both sides did **not** bring back the right trackball (was “halfway working,” mainly jumping around, before). We now know 4ms can be used without causing issues; it was not why the right trackball totally stopped.

**3. force-awake and 4ms combinations (process of elimination)**  
Test combinations of force-awake and force-awake-4ms-mode on both sides to see if any combo causes one side to stop working. Matrix: left (force-awake on/off, 4ms on/off) × right (force-awake on/off, 4ms on/off). Document which combos work with both trackballs.

**4. Layer-based speed (2× on specific layers)**  
Both trackballs stay at current low speed by default. On specific layers, override listeners so both run at **twice the speed**: pointer goes twice as far for same input, scroll goes twice as fast for same input. Use layer-specific input-processor overrides (e.g. zip_xy_scaler 2 1 for pointer, zip_scroll_scaler 2 40 for scroll on that layer).

**5. Single trackball, layer swaps pointer ↔ scroll (optional)**  
Experiment with using only one trackball: default = pointer (or scroll); on a second layer the same trackball switches to scroll (or pointer). Same keyboard works for left- or right-handed use (one side can be “mouse hand”). Unlikely to adopt long-term but fun to try.

**More ideas (dual trackball split)**

- **Swap roles per layer:** On one layer, left = pointer / right = scroll (opposite of default). Good for “mouse on left” or left-handed preference.
- **Precision vs speed layers:** One layer = 0.5× sensitivity (precision), another = 2× (fast). Complements the 2×-on-layer idea above.
- **Temporary layer on trackball use:** Use `zip_temp_layer` so touching a trackball temporarily activates a layer (e.g. layer 3 for 500 ms while moving). Good for different key bindings or actions while pointing/scrolling. (Already in keymap as commented-out example.)
- **Dual trackball for 2D pan:** On a “canvas” layer, one trackball = horizontal pan, one = vertical pan (or X/Y of a canvas). Fun for design/art apps.
- **Scroll direction or axis per layer:** On some layers, map one trackball to horizontal scroll only, or swap scroll axes, via scroll transform processors.
