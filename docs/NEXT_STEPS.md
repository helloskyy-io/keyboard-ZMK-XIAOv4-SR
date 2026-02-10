# Next steps: research and goals

This doc captures what we learned and concrete next steps (no coding yet). Use it to decide priorities and then implement.

---

## 1. How far back is our ZMK version? How many steps forward?

**Short answer: we are on the latest stable ZMK release, not “far back.”**

| ZMK release | Date | Notes |
|-------------|------|--------|
| v0.1.0 | 2024-11-29 | First versioned release |
| v0.2.0 | 2025-03-01 | Toggle mode, mouse/scroll, display config, etc. |
| v0.2.1 | 2025-03-02 | Bug fixes |
| **v0.3.0** | **2025-08-01** | **What we’re pinned to** – full-duplex wired split, nice!view profile status, BLE/profile fixes, pointing/split fixes |
| (no v0.4 yet) | — | `main` is ahead with Zephyr 4.1 work; no v0.4 tag yet |

So we’re **one step “back” from `main`** (which is bleeding edge). v0.3 is **cutting edge stable**.

**Useful things in v0.3 (we already have):** Full-duplex wired split, nice!view improvements, mouse/scroll, pointing fixes, BLE/profile address by index.

**If we moved to `main` later:** Zephyr 4.1, new boards, Hardware Model v2 – but we’d need the “board fix” (e.g. `xiao_ble` + overlay) and possibly badjeff `zmk-0.4` branch.

**Recommendation:** Stay on v0.3 for now (cutting edge, not bleeding edge). When there is a **v0.4 tag** and it’s considered a **stable release**, we can plan a move to v0.4 at that time.

---

## 2. Disable sleep / idle when keyboard is always plugged in

**Goal:** Keyboards are **never** wireless, never on battery; you want **no** sleep or idle effects. There must be a way to totally disable sleep for testing if nothing else.

### Where is `ZMK_SLEEP=n`?

**Yes – it’s in the right place.** We have:

- **`config ZMK_SLEEP`** with **`default n`** in **`boards/shields/skreecustom/Kconfig.defconfig`**.

ZMK loads shield defaults from `boards/shields/*/Kconfig.defconfig`, so that file is the correct place for this keyboard’s defaults. Deep sleep is therefore off for the skreecustom shield.

### Idle timeout – “value that high was not permitted”

- In **ZMK v0.3.0** source, **`CONFIG_ZMK_IDLE_TIMEOUT`** is a plain **`int`** with **no `range`** in Kconfig – so the build system does not enforce a maximum in code.
- If something previously rejected a very high value, it may have been:
  1. **Keymap Editor** or another tool that validates/edits config.
  2. **An older ZMK** that had a `range` in Kconfig.
  3. **menuconfig** or a config UI that caps displayed/entered values.

**What we’re doing:**

1. **Experiment with 24-hour idle:** Set **`CONFIG_ZMK_IDLE_TIMEOUT=86400000`** (24 hours in ms) in **`skreecustom_left.conf`** and **`skreecustom_right.conf`**. This is the only change in the first push so the build rebuilds and we can verify behavior.
2. **If it’s rejected:** Capture the **exact** error (build log or “not permitted” message and where it appears). Then we can check ZMK v0.3 and the Keymap Editor for any validation.
3. **“Totally disable” in ZMK today:** There is no “idle disabled” switch. The only knobs are: **deep sleep off** (`ZMK_SLEEP=n` – we have this) and **idle timeout as high as allowed**. So “totally disable” = keep deep sleep off + set idle timeout to the highest value the stack accepts.

---

## 3. Trackballs: force-awake and pointer jump (scroll vs pointer)

**Current setup (from our overlays):**

- **Left** = `trackball_central` → **scroll** (routed through `zip_xy_to_scroll_mapper` in `skreecustom.dtsi`). **No `force-awake`** on the left.
- **Right** = `trackball_peripheral` → **pointer** (cursor). **`force-awake`** is set on the right.

**What we tried before:** When **force-awake was added on both sides**, the **mouse pointer jumped** when first touching the **left (scroll)** trackball. That was undesirable, so we **reverted** and kept force-awake only on the right.

**Why that might happen:**  
The scroll trackball still reports X/Y (REL_X/REL_Y) before the input pipeline turns it into scroll. With `force-awake` on that sensor, timing or accumulation might change so that the first touch sends a burst of X/Y that is interpreted as pointer movement (or leaks into the pointer path). So **force-awake on the scroll (left) side can cause pointer jump**.

**Revised plan:**

- **Revisit force-awake on both sides** as an option (we previously reverted due to pointer jump when touching the left trackball; we may try again and see if behavior or config has changed).
- **Use `force-awake-4ms-mode`** for better tracking – sensor tracking leaves a lot to be desired; 4 ms (250 Hz) should help. Apply where we use force-awake (right now: right/pointer; if we re-enable on left: both).
- If pointer jump returns when force-awake is on both, we can fall back to force-awake + 4ms on the right (pointer) only.

### What is `force-awake-4ms-mode`?

From the badjeff README:

- **`force-awake-4ms-mode`** applies **only when `force-awake` is set**.
- It changes the **sampling interval** from **8 ms** (default) to **4 ms** → **250 Hz** reporting when the sensor is force-awake.
- **Use case:** “Apply this mode if you need **250 Hz with direct USB connection**” for a smoother cursor.

So: **4 ms = 250 Hz polling** of the trackball when it’s kept awake; 8 ms = 125 Hz. For better tracking (these sensors leave a lot to be desired), we will use the 4 ms option.

---

## 4. badjeff and newer ZMK / rotation (“north” alignment)

**Newer ZMK / zmk-0.4:**  
There’s a **`zmk-0.4`** branch for future ZMK 0.4 (alt compatible string and config). No extra features we need right now; we stay on **zmk-0.3** until we move to v0.4.

**Rotation / “north” in degrees:**  
You want to **fine-tune the relative north position** of the trackballs (independent of physical mount) so “up” aligns with how you work and to offset rotation in the mount.

- The **badjeff driver** (and README) does **not** offer **rotation in degrees**.
- It replaced the old `CONFIG_PMW3610_ORIENTATION_*` with **devicetree** options:
  - **`swap-xy`** – swap X and Y axes.
  - **`invert-x`** – invert X.
  - **`invert-y`** – invert Y.

So we only get **eight orientations** (0°, 90°, 180°, 270° and their mirrors), not arbitrary angles (e.g. 15° or 45°). That’s enough to align “north” in 90° steps and fix a wrong physical rotation by 90° or 180°, but **not** for fine angular adjustment.

**If you need true “rotation in degrees”:**

- Would require **driver or ZMK changes** (e.g. a rotation matrix or an angle property in the driver or in ZMK’s pointing stack). Not in the current badjeff driver.
- Worth checking **badjeff issues/PRs** and **ZMK pointing docs** for any “rotation” or “orientation angle” feature request or workaround; we didn’t find one in a quick look.

**Concrete next steps:**

- Use **`swap-xy`**, **`invert-x`**, **`invert-y`** in the trackball node(s) in our overlays to get the closest “north” we can (90° steps).
- **Submit an issue in badjeff** (zmk-pmw3610-driver) requesting **rotation in degrees** (or similar) for fine-tuning “north” independent of physical mount. Link the issue here once created.

---

## 5. Plan summary (revised)

| Goal | What we know | Next step |
|------|----------------|-----------|
| **Stay on v3 / move to v4** | Agreed: stay on v0.3; move once v0.4 is considered the stable option. | Stay on v0.3; when **v0.4 is released and stable**, plan a move to v0.4. |
| **Idle timeout** | `ZMK_SLEEP=n` is in the correct place. Idle has no “off” switch; only timeout. | **Done (this push):** Set **`CONFIG_ZMK_IDLE_TIMEOUT=86400000`** (24 h) in **`skreecustom_left.conf`** and **`skreecustom_right.conf`**. Push so it rebuilds; verify. If rejected, capture the exact error. |
| **Trackballs / force-awake** | force-awake on both previously caused pointer jump when touching left (scroll); we reverted. Tracking is really terrible. | **Revisit force-awake on both sides** as an option (try again; behavior may differ). Use **`force-awake-4ms-mode`** for better tracking (250 Hz). If pointer jump returns, fall back to force-awake + 4ms on right (pointer) only. |
| **force-awake-4ms-mode** | 4 ms = 250 Hz when force-awake is on; 8 ms default = 125 Hz. Better for direct USB and tracking. | Use **4 ms option** where we use force-awake (right now: right; if we re-enable on left: both) to improve tracking. |
| **Rotation / north** | badjeff gives **swap-xy**, **invert-x**, **invert-y** only → 8 orientations (90° steps). No rotation in degrees. | Use swap/invert for now. **Submit an issue in badjeff** (zmk-pmw3610-driver) requesting rotation in degrees (or similar) for fine-tuning “north”; link it here once created. |
| **badjeff and v0.4** | zmk-0.3 = current; zmk-0.4 = for future ZMK 0.4. | Stay on zmk-0.3; when we upgrade ZMK to v0.4, switch driver to zmk-0.4 and update to “alt” names. |

**Implementation order:** (1) **This push:** 24 h idle only – push and let it rebuild. (2) **Next:** Revisit force-awake on both + add force-awake-4ms-mode for better tracking. (3) **Separately:** Submit badjeff issue for rotation in degrees.
