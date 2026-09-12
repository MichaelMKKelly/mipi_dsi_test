# MIPI DSI v3 Fix — Report

**Device:** EspControl 10inch V3 (`guition-esp32-p4-jc8012p4a1-v3`)
**Silicon:** ESP32-P4, rev 3.2 production
**Component changed:** `mipi_dsi` (local override of the stock ESPHome display component)
**Date:** 2026-09-12

---

## TL;DR

The v3 panel would boot, enumerate over USB, print a few log lines, then crash and reboot in a loop. The crash was in the display (MIPI DSI) driver, not in our app code.

The stock ESPHome `mipi_dsi` component hardcodes a specific reference clock for the display PHY that **only exists on the older ESP32-P4 chips**. The newer rev 3.2 silicon on the v3 panel doesn't have that clock option, so the driver hit a "this should never happen" error and aborted.

The fix is a one-line change: instead of forcing that old clock, we leave the clock source *unassigned* so the chip's own driver picks the correct clock for the silicon revision. On rev 3.0+ chips (like the v3 panel) it picks **XTAL**, which is valid.

---

## The symptom

After flashing, the device:

1. Powered on and enumerated over USB.
2. Printed a few early log lines.
3. Crashed with:

   ```
   abort() was called at PC 0x4015d941 on core 1
   ```

4. Rebooted and repeated the loop.

Symbolizing the crash address (`0x4015d941`) against the build's ELF pointed at:

```
_mipi_dsi_ll_set_phy_pllref_clock_source   (mipi_dsi_ll.h:218, `default: abort()`)
```

That is the low-level function that configures which clock feeds the DSI PHY's PLL. The `default: abort()` means it was handed a clock value it doesn't recognize.

---

## Root cause

The DSI PHY needs a reference clock to generate its output frequency. On the ESP32-P4 there are several possible sources (XTAL, APLL, CPLL, SPLL, MPLL, and a legacy 20 MHz PLL called `PLL_F20M`).

The stock ESPHome component sets this in the DSI bus config:

```cpp
.phy_clk_src = MIPI_DSI_PHY_CLK_SRC_DEFAULT,
```

That macro resolves, in the ESP-IDF headers, to the **legacy** source:

```
MIPI_DSI_PHY_CLK_SRC_DEFAULT
  = MIPI_DSI_PHY_PLLREF_CLK_SRC_DEFAULT_LEGACY
  = SOC_MOD_CLK_PLL_F20M          // the legacy 20 MHz PLL
```

On **rev 3.0+** silicon, the HAL function that applies this setting only knows how to handle `XTAL / APLL / CPLL / SPLL / MPLL`. It does **not** handle `PLL_F20M` (that clock path was removed/changed in the newer silicon). So when the stock component passes `PLL_F20M`, the function falls through to `default: abort()` — the crash we saw.

In short: the stock component assumes the old silicon and forces a clock the new silicon can't use.

---

## The fix

The ESP-IDF DSI driver already has a built-in fallback: **if the clock source is left unassigned (`0`), the driver picks the correct default for the silicon revision itself.**

From `esp_lcd_mipi_dsi_bus.c`:

- `phy_clk_src == 0` (unassigned) → driver chooses:
  - pre-v3 silicon → `DEFAULT_LEGACY` (`PLL_F20M`)
  - rev 3.0+ silicon → `DEFAULT` (`XTAL`)
- `phy_clk_src != 0` → driver uses exactly what you gave it (no fallback).

So the correct behavior is to **not** force a value and let the driver decide. The change in `mipi_dsi.cpp` (`MipiDsi::setup()`):

```cpp
esp_lcd_dsi_bus_config_t bus_config = {
    .bus_id = 0,
    .num_data_lanes = this->lanes_,
    // 0 = "unassigned": let the ESP-IDF DSI driver pick the PHY PLL reference
    // clock that matches the silicon revision. The stock ESPHome value
    // (MIPI_DSI_PHY_CLK_SRC_DEFAULT) is the legacy PLL_F20M source, which the
    // ESP32-P4 rev 3.0+ HAL does not handle and aborts on at boot. With 0, the
    // driver falls back to XTAL on rev 3.0+ silicon (PLL_F20M on pre-v3).
    .phy_clk_src = static_cast<mipi_dsi_phy_pllref_clock_source_t>(0),
    .lane_bit_rate_mbps = this->lane_bit_rate_,
};
```

A log line was also added so we can confirm at runtime that this override is the one running:

```cpp
ESP_LOGCONFIG(TAG, "Running Setup (v3 local mipi_dsi override: phy_clk_src=0)");
```

### Why `static_cast<...>(0)` and not just `0`

`phy_clk_src` is a C enum type (`mipi_dsi_phy_pllref_clock_source_t`). C++ won't implicitly convert a plain integer literal to an enum, so the value is cast explicitly to the enum type. The value is still `0` (unassigned) — the cast just satisfies the compiler.

---

## Why it works

- On the v3 panel (rev 3.2), the driver sees `phy_clk_src == 0` and selects **XTAL**, a valid reference clock on rev 3.0+ silicon.
- The HAL function receives a value it recognizes, so it no longer hits `default: abort()`.
- The DSI PHY initializes, the panel comes up, and the boot loop is gone.
- Because the driver still applies its own revision-aware default, this is the *intended* way to configure the clock — we're just no longer overriding it with a value that's invalid on the new silicon.

---

## How it's wired in

The change lives in a **local copy** of the component at `components/mipi_dsi/`, used only by the v3 build via `external_components` in `devices/guition-esp32-p4-jc8012p4a1-v3/device/device.yaml`:

```yaml
external_components:
  # ... (existing git-sourced components) ...
  # Local override of the stock ESPHome mipi_dsi display component.
  - source:
      type: local
      path: ../components
    components: [mipi_dsi]
```

This means:

- **Only the v3 build** uses the modified component.
- **v2 and every other panel** keep using the stock built-in `mipi_dsi`, so nothing else is affected.
- The local copy is a full copy of the stock component with the single change above — easy to diff and easy to drop if upstream ESPHome fixes this.

---

## How to verify

1. **Compile** — the v3 factory firmware builds cleanly.
2. **Confirm the override is active** — the build's generated source contains the fix, and at runtime the serial log shows:
   ```
   Running Setup (v3 local mipi_dsi override: phy_clk_src=0)
   ```
3. **Confirm the crash is gone** — flash `to_test/firmware.factory.bin` to the v3 panel. The display should come up and the device should stay connected instead of rebooting.

---

## Scope / impact

- **Affected:** v3 panel only (rev 3.0+ silicon).
- **Not affected:** v2 and all other panels (they still use the stock component and the legacy clock, which is correct for their silicon).
- **Risk:** low. The change only affects which reference clock the DSI PHY uses, and it defers to the driver's own revision-aware default rather than hardcoding a value.
