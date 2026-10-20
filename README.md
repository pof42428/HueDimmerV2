# 💡 Advanced & Fully Optional Adaptive Hue Dimmer v2 (Zigbee2MQTT) Blueprint for Home Assistant

An advanced, flexible Home Assistant automation blueprint for the **[Philips Hue Dimmer Switch v2 (`929002398602` / `RWL022`)](https://www.zigbee2mqtt.io/devices/929002398602.html)** via **Zigbee2MQTT**.

This blueprint controls any number of lights ($n$ entities, light groups, or entire areas) with support for **dual-mode Up/Down controls** (Brightness vs. Kelvin Color Temperature), **sun-based adaptive lighting**, **visual RGB feedback**, and **custom safe min/max bounds**.

---

## ✨ Features

- **Optional Entities & Smart Fallbacks:** Every single input entity (helpers, sensors, min/max bounds) is completely optional. If left blank, the blueprint gracefully skips that functionality or uses safe default values without throwing schema or template errors.
- **Dual-Mode Up/Down Dimming:** Toggle the function of the Up/Down physical buttons between **Brightness Control** (+/- 10%) and **Color Temperature Control** (+/- 500K).
- **Adaptive Day/Night Lighting:**
  - **Daytime (`sun.sun` = `above_horizon`):** Dynamically calculates target brightness and color temperature from your sun/solar sensors, bounded between your safe limits.
  - **Nighttime (`sun.sun` = `below_horizon`):** Falls back to comfortable minimum brightness and warm color temperature limits to protect your eyes.
- **Visual RGB Indicator:** Holding the Off button toggles Adaptive Mode on/off and briefly flashes your lights **Green (Enabled)** or **Red (Disabled)** before restoring your light settings.
- **Real-Time Sun Tracking:** Updates active lights dynamically whenever your solar Kelvin sensor updates, provided Adaptive Mode is enabled.

---

## 🎮 Button Mapping

| Button Action | Physical Button | Function |
| :--- | :--- | :--- |
| **On Press** | `on_press_release` | **Turn On / Off:** Turns on lights to the calculated adaptive or default values and turns **Off** Adaptive Mode to respect manual use. If already on, turns off lights and resets mode to Brightness control. |
| **Up Press** | `up_press_release` | **Increase:** Step up Brightness by +10% (if in Brightness mode) OR increase Color Temperature by +500K / cooler white (if in Kelvin mode). |
| **Down Press** | `down_press_release` | **Decrease:** Step down Brightness by -10% (if in Brightness mode) OR decrease Color Temperature by -500K / warmer white (if in Kelvin mode). |
| **Off Press** | `off_press_release` | **Mode Toggle:** Toggles the control mode helper between **Brightness Control** (ON) and **Kelvin Control** (OFF). |
| **Off Hold** | `off_hold_release` | **Adaptive Toggle:** Toggles live adaptive sun tracking. Flashes green when enabled, red when disabled, then restores light settings. |

---

## 🧩 Understanding Blueprint Entities & Inputs

Below is a detailed breakdown of all entities configured in the blueprint and how they function:

### 1. Device & Target Inputs (Core Controls)

* **`remote_device_id` (Required):**
  * **Domain:** `device` (`integration: mqtt`)
  * **Role:** The physical Philips Hue Dimmer Switch v2 device paired via Zigbee2MQTT. The blueprint listens to device triggers (`on_press_release`, `up_press_release`, etc.) originating from this hardware unit.
* **`target_light` (Required):**
  * **Domain:** `target` (`light`)
  * **Role:** The target entity, group of lights, or room area you want to control. All dimming, switching, and color updates are dispatched directly to this target.

---

### 2. Helper Inputs (`input_boolean`)

* **`control_mode_boolean` (Optional):**
  * **Domain:** `input_boolean`
  * **Role:** Keeps track of what the physical **Up** and **Down** buttons control:
    * **`on` (Default/Fallback if blank):** Up/Down buttons adjust **Brightness** (+/- 10%).
    * **`off`:** Up/Down buttons adjust **Color Temperature** (+/- 500 Kelvin).
  * **How it works:** Pressing the **Off button (Short Press)** toggles this boolean between `on` and `off`.
* **`adaptive_mode_boolean` (Optional):**
  * **Domain:** `input_boolean`
  * **Role:** Toggles live adaptive sun-tracking on or off.
    * **`on`:** The lights dynamically adjust whenever the `sun_kelvin_sensor` updates.
    * **`off`:** Live tracking is paused, preventing sun updates from overriding manual adjustments.
  * **How it works:** Pressing the **On button** sets this helper to `off` so manual remote presses are respected. **Holding the Off button** toggles this helper `on`/`off` with visual RGB flashing.

---

### 3. Sensor Inputs (`sensor`)

* **`sun_kelvin_sensor` (Optional):**
  * **Domain:** `sensor`
  * **Role:** Provides the current target color temperature in Kelvin based on sun elevation or time of day (e.g., `sensor.sun_kelvin`).
  * **Fallback:** If blank or unavailable, defaults to a static **`2700K`**.
* **`sun_brightness_sensor` (Optional):**
  * **Domain:** `sensor`
  * **Role:** Provides the target brightness percentage (0–100%) based on solar position or outdoor light levels (e.g., `sensor.sun_brightness`).
  * **Fallback:** If blank or unavailable, defaults to a static **`50%`**.

---

### 4. Boundary Limit Inputs (`sensor` or `input_number`)

These inputs define safe minimum and maximum limits to protect against uncomfortable light levels. You can select an `input_number` helper or a custom `sensor`. **If left blank, hardcoded fallback defaults are used.**

* **`min_brightness_entity` (Optional):**
  * **Role:** Defines the lowest allowed brightness level (%).
  * **Fallback:** `20%`
* **`max_brightness_entity` (Optional):**
  * **Role:** Defines the highest allowed brightness level (%).
  * **Fallback:** `100%`
* **`min_kelvin_entity` (Optional):**
  * **Role:** Defines the warmest allowed color temperature (in Kelvin, e.g., 2700K). Used during nighttime turn-ons.
  * **Fallback:** `2700K`
* **`max_kelvin_entity` (Optional):**
  * **Role:** Defines the coolest allowed color temperature (in Kelvin, e.g., 5500K / 6000K).
  * **Fallback:** `5500K`

---

## ☀️ Day vs. Night Logic Explained

The blueprint checks the state of `sun.sun` (`above_horizon` vs `below_horizon`) to decide how values are calculated:

$$\text{Daytime Target} = \min(\max(\text{Sun Sensor Value}, \text{Min Limit}), \text{Max Limit})$$
