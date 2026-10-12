# Home Assistant PV Excess EV Charging Package

This is a multi-file Home Assistant automation package for solar-aware EV charging with time-based hysteresis and fully user-tunable values.

## File layout

```text
config/
├─ configuration.yaml                              # package includes (merge into your own file)
├─ packages/
│  ├─ pv_ev_charger/
│  │  ├─ helpers.yaml                              # input_number, input_text, input_datetime, input_boolean
│  │  ├─ templates.yaml                            # template sensors (solar power, target current)
│  │  └─ automations.yaml                          # step up/down control automation
│  └─ examples/
│     └─ battery-full-stop-automation.yaml         # optional battery-full stop automation
└─ dashboards/
   └─ pv_ev_charge_controller_card.yaml            # dashboard card
```

| File | Purpose |
|------|---------|
| `packages/pv_ev_charger/helpers.yaml` | All user-tunable helpers, entity-name settings, state helpers and the safety override |
| `packages/pv_ev_charger/templates.yaml` | `sensor.pv_solar_power` and `sensor.pv_target_charge_current` |
| `packages/pv_ev_charger/automations.yaml` | `PV EV Charger - Update Current Step` (step up/down with hysteresis) |
| `packages/examples/battery-full-stop-automation.yaml` | Optional: stop charging when the charger current drops to 0 A for 1 minute |
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

3. (Optional) Copy `packages/examples/battery-full-stop-automation.yaml` and include it the same way, for example:

```yaml
    pv_ev_charger_battery_full: !include packages/examples/battery-full-stop-automation.yaml
```

4. Check the configuration (Developer Tools > YAML > Check Configuration), then restart Home Assistant.
5. Adjust the tunable values in the UI under Settings > Devices & Services > Helpers.

> **Note:** If your Home Assistant config already loads another PV EV charger package (for example an older `packages/pv_ev_charger.yaml`), remove or disable it first. Both define overlapping helpers, templates and automations and will conflict.

## User-Tunable Values

All thresholds and timings can be adjusted in the Home Assistant UI without editing YAML.

### Power Thresholds (Watts)
- `input_number.pv_step_1_threshold` - Minimum power to start charging (default: 1000 W)
- `input_number.pv_step_2_threshold` - Step 2 threshold (default: 1500 W)
- `input_number.pv_step_3_threshold` - Step 3 threshold (default: 2500 W)
- `input_number.pv_step_4_threshold` - Step 4 threshold (default: 3000 W)

### Charge Current Levels (Amps)
- `input_number.pv_step_1_current` - Current at step 1 (default: 6 A)
- `input_number.pv_step_2_current` - Current at step 2 (default: 8 A)
- `input_number.pv_step_3_current` - Current at step 3 (default: 10 A)
- `input_number.pv_step_4_current` - Current at step 4 (default: 15 A)

### Hysteresis Timing (Minutes)
- `input_number.pv_step_up_delay_minutes` - Time to wait before stepping up (default: 3 min)
- `input_number.pv_step_down_delay_minutes` - Time to wait before stepping down (default: 2 min)

### System Configuration
- `input_text.pv_solar_power_sensor` - Your solar power sensor (default: `sensor.pv_power_watts`)
- `input_text.pv_charger_current_entity` - Your charger current control (default: `number.ev_charger_current_amps`)
- `input_text.pv_charger_phase_1_current` - Your charger phase 1 current sensor (default: `sensor.ev_ac_charging_control_box_current_phase_1`)
- `input_text.pv_ev_connected_sensor` - Your EV connected sensor (default: `binary_sensor.ev_connected`)

### State Helpers (managed by the automations)
- `input_text.pv_current_step` - Current step label (`idle` or `step_<amps>`)
- `input_datetime.pv_last_step_change` - Timestamp of the last step change
- `input_boolean.pv_charge_safety_override` - When **on**, the automations are disabled

## How it works

1. `sensor.pv_solar_power` reads the sensor named in `input_text.pv_solar_power_sensor`.
2. `sensor.pv_target_charge_current` maps solar power to a charge current using the step thresholds (below step 1 = 0 A).
3. The `PV EV Charger - Update Current Step` automation runs when the target, thresholds, currents or delays change. It only runs if the safety override is off and the EV connected sensor is `on`.
4. It **steps up** only if the target is higher than the current setting and at least `pv_step_up_delay_minutes` have passed since the last change. It **steps down** only if the target is lower and at least `pv_step_down_delay_minutes` have passed.
5. Each change updates the charger, `pv_current_step` and `pv_last_step_change`.

### Important cutoff behavior
- If PV power drops below `input_number.pv_step_1_threshold`, the target current becomes `0 A`.
- With the default guide values, this means anything below **1500 W** stops charging.
- Because the main automation already handles the low-power cutoff, the battery-full stop automation is **optional and generally not recommended** for this configuration.

## How to use

1. **Verify entity names**: Check that your solar power, charger current, EV connected and charger phase current entities match the defaults. If not, update them in the `input_text` helpers.
2. **Adjust thresholds**: Set the power thresholds to match your solar system size and charger capabilities.
3. **Set current levels**: Configure the charge current for each step (0 A = no charging).
4. **Fine-tune timing**: Adjust the up/down delays based on your local weather patterns.
5. **Monitor via dashboard**: Add the card from `dashboards/pv_ev_charge_controller_card.yaml`.
6. **Battery-full stop automation**: Not needed for the default setup because charging stops when solar drops below the minimum threshold.

## Low-Power Pause and Resume Behavior

If PV power drops below the minimum charging threshold during the active time window, charging pauses by setting the charger current to `0 A`.

The automation remains active while the window is open, so if solar output rises above the threshold again before the window ends, charging resumes automatically.

Charging only stops permanently when the configured time window ends.

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

If you changed the entity names in the helpers, update the card to match.

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
- Check that `input_text.pv_ev_connected_sensor` is correctly configured
- Ensure the EV connected sensor is in the "on" state when you expect charging
- Ensure `input_boolean.pv_charge_safety_override` is "off"

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

To change how solar power maps to a target current, edit `packages/pv_ev_charger/templates.yaml`.

---

A multi-file Home Assistant package for solar-aware EV charging with tunable hysteresis and automatic pause/resume during the active time window.
