# Home Assistant PV Excess EV Charging Package

This is a multi-file Home Assistant automation package for solar-aware EV charging with time-based hysteresis, a configurable daily charge window, and fully user-tunable values.

The automation only controls the charger **inside the charge window**. Outside the window it does nothing, so the charger is released for manual use.

## File layout

```text
config/
|- configuration.yaml                              # package includes (merge into your own file)
|- packages/
|  |- pv_ev_charger/
|  |  |- helpers.yaml                              # input_number, input_text, input_datetime, input_boolean
|  |  |- templates.yaml                            # solar power, target current, charge window sensors
|  |  `- automations.yaml                          # step control automation (window-gated)
|  `- examples/
|     `- battery-full-stop-automation.yaml         # optional, NOT recommended (see below)
`- dashboards/
   `- pv_ev_charge_controller_card.yaml            # dashboard card
```

| File | Purpose |
|------|---------|
| `packages/pv_ev_charger/helpers.yaml` | All user-tunable helpers, entity-name settings, charge window times, state helpers and the safety override |
| `packages/pv_ev_charger/templates.yaml` | `sensor.pv_solar_power`, `sensor.pv_target_charge_current` and `binary_sensor.pv_charge_window_active` |
| `packages/pv_ev_charger/automations.yaml` | `PV EV Charger - Update Current Step` (only acts inside the charge window) |
| `packages/examples/battery-full-stop-automation.yaml` | Optional reference only. Not part of the default install |
| `dashboards/pv_ev_charge_controller_card.yaml` | Entities card to monitor the system |

## Installation

1. Copy the `packages/pv_ev_charger/` folder into your Home Assistant `packages/` directory (create it if it doesn't exist).
2. Add the includes to your `configuration.yaml`. **Merge** this into your existing file rather than replacing it:

```yaml
homeassistant:
  packages:
    pv_ev_charger_helpers: !include packages/pv_ev_charger/helpers.yaml
    pv_ev_charger_templates: !include packages/pv_ev_charger/templates.yaml
    pv_ev_charger_automations: !include packages/pv_ev_charger/automations.yaml
```

3. Check the configuration (Developer Tools > YAML > Check Configuration), then restart Home Assistant.
4. Adjust the tunable values in the UI under Settings > Devices & Services > Helpers.

> **Note:** If your Home Assistant config already loads another PV EV charger package (for example an older `packages/pv_ev_charger.yaml`), remove or disable it first. Both define overlapping helpers, templates and automations and will conflict.

## User-Tunable Values

All thresholds, timings and the charge window can be adjusted in the Home Assistant UI without editing YAML.

### Power Thresholds (Watts)
- `input_number.pv_step_1_threshold` - Minimum power to charge. Below this the charger is set to 0 A (default: 1500 W)
- `input_number.pv_step_2_threshold` - Step 2 threshold (default: 2000 W)
- `input_number.pv_step_3_threshold` - Step 3 threshold (default: 2500 W)
- `input_number.pv_step_4_threshold` - Step 4 threshold (default: 3000 W)

Keep the thresholds in ascending order (step 1 < step 2 < step 3 < step 4).

### Charge Current Levels (Amps)
- `input_number.pv_step_1_current` - Current at step 1 (default: 6 A)
- `input_number.pv_step_2_current` - Current at step 2 (default: 8 A)
- `input_number.pv_step_3_current` - Current at step 3 (default: 10 A)
- `input_number.pv_step_4_current` - Current at step 4 (default: 15 A)

### Hysteresis Timing (Minutes)
- `input_number.pv_step_up_delay_minutes` - Time to wait before stepping up (default: 3 min)
- `input_number.pv_step_down_delay_minutes` - Time to wait before stepping down (default: 2 min)

### Charge Window
- `input_datetime.pv_charge_window_start` - Time the automation starts controlling the charger (default: 09:00)
- `input_datetime.pv_charge_window_end` - Time the automation stops controlling the charger (default: 16:00)

Windows that cross midnight are supported (for example start 22:00, end 06:00).

### System Configuration
- `input_text.pv_solar_power_sensor` - Your solar power sensor (default: `sensor.pv_power_watts`)
- `input_text.pv_charger_current_entity` - Your charger current control (default: `number.ev_charger_current_amps`)
- `input_text.pv_charger_phase_1_current` - Your charger phase 1 current sensor (default: `sensor.ev_ac_charging_control_box_current_phase_1`)
- `input_text.pv_ev_connected_sensor` - Your EV connected sensor (default: `binary_sensor.ev_connected`)

### State Helpers (managed by the automation)
- `input_text.pv_current_step` - Current step label (`idle` or `step_<amps>`)
- `input_datetime.pv_last_step_change` - Timestamp of the last step change
- `input_boolean.pv_charge_safety_override` - When **on**, the automation is disabled

## How it works

1. `sensor.pv_solar_power` reads the sensor named in `input_text.pv_solar_power_sensor`.
2. `sensor.pv_target_charge_current` maps solar power to a charge current using the step thresholds (below step 1 = 0 A).
3. `binary_sensor.pv_charge_window_active` is `on` between the configured window start and end times.
4. The `PV EV Charger - Update Current Step` automation runs when the target, window state, thresholds, currents or delays change. It only acts if the safety override is off, the charge window is active and the EV connected sensor is `on`.
5. It **steps up** only if the target is higher than the current setting and at least `pv_step_up_delay_minutes` have passed since the last change. It **steps down** only if the target is lower and at least `pv_step_down_delay_minutes` have passed.
6. Each change updates the charger, `pv_current_step` and `pv_last_step_change`.

## Low-Power Pause and Resume Behavior

If PV power drops below `pv_step_1_threshold` (default 1500 W) during the active charge window, charging pauses by setting the charger current to `0 A` (after the step-down delay).

The automation stays active while the window is open, so if solar output rises above the threshold again before the window ends, charging resumes automatically (after the step-up delay).

## Outside the Charge Window (manual charging)

Outside the charge window the automation is completely hands-off:

- it does not send any commands to the charger
- it does not set the current to 0 A when the window closes
- you can charge manually and change the charger current yourself

Note that the charger keeps whatever current was last set when the window closed (for example 6 A or 15 A). Set it yourself if you want a different value for manual charging.

When the window opens again, the automation takes over and adjusts the current to match solar output.

If you want to stop automatic control during the window (for example to charge manually at midday), turn on `input_boolean.pv_charge_safety_override`.

## How to use

1. **Verify entity names**: Check that your solar power, charger current, EV connected and charger phase current entities match the defaults. If not, update them in the `input_text` helpers.
2. **Set the charge window**: Set `pv_charge_window_start` and `pv_charge_window_end`.
3. **Adjust thresholds**: Set the power thresholds to match your solar system size and charger capabilities.
4. **Set current levels**: Configure the charge current for each step (0 A = no charging).
5. **Fine-tune timing**: Adjust the up/down delays based on your local weather patterns.
6. **Monitor via dashboard**: Add the card from `dashboards/pv_ev_charge_controller_card.yaml`.

## Battery-Full Stop Automation (not recommended)

The battery-full stop automation is **not** part of the default setup. The control automation already sets the charger to 0 A when solar drops below the minimum threshold, and a separate stop-at-zero-current trigger could end charging early and prevent it resuming when solar recovers within the window.

`packages/examples/battery-full-stop-automation.yaml` is kept only as a reference. Do not include it in `configuration.yaml` unless you specifically want that behavior.

## Dashboard Card

Add the contents of `dashboards/pv_ev_charge_controller_card.yaml` to a dashboard. This is a Lovelace card, not a package include.

```yaml
type: entities
title: PV EV Charge Controller
entities:
  - entity_id: sensor.pv_power_watts
    name: Solar Power
  - entity_id: number.ev_charger_current_amps
    name: Charge Current
  - entity_id: input_text.pv_current_step
    name: Current Step
  - entity_id: binary_sensor.ev_connected
    name: EV Connected
  - entity_id: sensor.ev_ac_charging_control_box_current_phase_1
    name: Charger Phase 1 Current
  - entity_id: input_datetime.pv_last_step_change
    name: Last Step Change
```

If you changed the entity names in the helpers, update the card to match. You can also add `binary_sensor.pv_charge_window_active`, `input_datetime.pv_charge_window_start` and `input_datetime.pv_charge_window_end` to see the window state.

## Troubleshooting

**Configuration check fails / package won't load:**
- Check indentation and that each `!include` path matches the file location
- Make sure no other package defines the same helpers, templates or automations

**Charger not responding:**
- Check that `input_text.pv_charger_current_entity` matches your actual charger entity
- Verify the charger entity supports the `number.set_value` service

**Solar sensor not reading:**
- Check that `input_text.pv_solar_power_sensor` matches your solar generation sensor
- Confirm the sensor is actively reporting values in Home Assistant

**Automation not triggering:**
- Check that `binary_sensor.pv_charge_window_active` is `on` (check the window start/end helpers)
- Check that `input_text.pv_ev_connected_sensor` is correctly configured
- Ensure the EV connected sensor is in the "on" state when you expect charging
- Ensure `input_boolean.pv_charge_safety_override` is "off"

**Charger changes current outside the window:**
- Check `pv_charge_window_start` / `pv_charge_window_end` are set as expected
- Check that no other automation or package is controlling the charger

**Charger stepping too fast/slow:**
- Adjust `input_number.pv_step_up_delay_minutes` and `input_number.pv_step_down_delay_minutes`
- Check the automation trace and logs for trigger timestamps

## Advanced: Modifying the automation logic

To change behavior beyond the tunable values, edit `packages/pv_ev_charger/automations.yaml`. The main automation is `alias: "PV EV Charger - Update Current Step"`.

Key variables:
- `target_current`: Charge current calculated from solar power (from `sensor.pv_target_charge_current`)
- `charger_entity`: Charger entity read from `input_text.pv_charger_current_entity`
- `current_current`: The charger's present current setting
- `minutes_since_change`: Minutes elapsed since the last step change
- `up_delay` / `down_delay`: Hysteresis delays from the helpers

To change how solar power maps to a target current, or how the charge window is evaluated, edit `packages/pv_ev_charger/templates.yaml`.

---

A multi-file Home Assistant package for solar-aware EV charging with tunable hysteresis, a configurable charge window, automatic pause/resume within the window, and no interference with the charger outside the window.
