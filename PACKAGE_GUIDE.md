# Home Assistant PV Excess EV Charging Package

This is a complete Home Assistant automation package for solar-aware EV charging with time-based hysteresis and fully user-tunable values.

## Installation

1. Copy `pv_ev_charger.yaml` into your `packages/` directory (create it if it doesn't exist)
2. Add this line to your `configuration.yaml`:

```yaml
homeassistant:
  packages:
    pv_ev_charger: !include packages/pv_ev_charger.yaml
```

3. Restart Home Assistant
4. Adjust the tunable values in the UI under Settings > Devices & Services > Helpers

## User-Tunable Values

All thresholds and timings can be adjusted in the Home Assistant UI without editing YAML:

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
- `input_text.pv_solar_power_sensor` - Name of your solar power sensor (default: `sensor.pv_power_watts`)
- `input_text.pv_charger_current_entity` - Name of your charger current control (default: `number.ev_charger_current_amps`)
- `input_text.pv_charger_phase_1_current` - Name of your charger phase 1 current sensor (default: `sensor.ev_ac_charging_control_box_current_phase_1`)
- `input_text.pv_ev_connected_sensor` - Name of your EV connected sensor (default: `binary_sensor.ev_connected`)

## How to use

1. **Verify entity names**: Check that your solar power, charger current, EV connected, and charger phase current sensors match the defaults. If not, update them in the helpers.

2. **Adjust thresholds**: Set the power thresholds to match your solar system size and charger capabilities.

3. **Set current levels**: Configure the charge current for each step (0 A = no charging).

4. **Fine-tune timing**: Adjust the up/down delay times based on your local weather patterns.

5. **Enable battery-full stop**: Deploy the automation from `examples/battery-full-stop-automation.yaml` to automatically stop charging when the battery reaches full capacity.

6. **Monitor via dashboard**: Use the provided dashboard card to watch the system in action.

## Battery-Full Stop Automation

If your EV doesn't expose battery SOC data, use the battery-full stop automation to automatically halt charging when the charger's current draw drops to zero for 1 minute. This indicates the battery is fully charged.

Copy the automation from `examples/battery-full-stop-automation.yaml` to your Home Assistant configuration. It monitors `sensor.ev_ac_charging_control_box_current_phase_1` and stops the charger when charging completes.

## Dashboard Card

Add this to your dashboard to monitor the system:

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

## Troubleshooting

**Charger not responding:**
- Check that `input_text.pv_charger_current_entity` matches your actual charger entity
- Verify the charger entity supports the `number.set_value` service

**Solar sensor not reading:**
- Check that `input_text.pv_solar_power_sensor` matches your solar generation sensor
- Confirm the sensor is actively reporting values in Home Assistant

**Automation not triggering:**
- Check that `input_text.pv_ev_connected_sensor` is correctly configured
- Ensure the EV connected sensor is in the "on" state when you expect charging

**Charger stepping too fast/slow:**
- Adjust `input_number.pv_step_up_delay_minutes` and `input_number.pv_step_down_delay_minutes`
- Check the logs for automation trigger timestamps

**Battery-full stop automation not working:**
- Verify `input_text.pv_charger_phase_1_current` is correctly set to your charger's phase 1 current sensor
- Check that the sensor is reporting 0 A when charging completes
- Ensure `input_boolean.pv_charge_safety_override` is in the "off" state (otherwise automations are disabled)

## Advanced: Modifying the automation logic

If you need to change the automation behavior beyond tunable values, edit the `pv_ev_charger.yaml` file directly. The main automation is `alias: "PV EV Charger - Control by Solar Power"`.

Key variables:
- `solar_power`: Current solar power reading
- `target_current`: Calculated charge current based on power
- `time_since_change`: Minutes elapsed since last step change
- `should_change`: Boolean indicating if a step change should occur

For the battery-full stop automation, the key trigger is the phase 1 current sensor dropping to 0 A and remaining there for 1 minute.

---

This package provides a complete, user-friendly Home Assistant solution for solar-aware EV charging with automatic battery-full detection.
