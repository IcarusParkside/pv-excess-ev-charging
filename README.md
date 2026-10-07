# Home Assistant PV Excess EV Charging

This project is designed for a Home Assistant setup where EV charging current is automatically adjusted based on available solar power generation. Rather than simple on/off control, this approach uses real-time solar production to set charger current in small stepped increments, creating a smooth and efficient charging profile.

## Why this fits Home Assistant

Home Assistant excels at:

- monitoring solar generation power (`sensor.pv_power_watts`)
- controlling EV charger current levels (`number.ev_charger_current_amps`)
- tracking EV connection state (`binary_sensor.ev_connected`)
- managing time-based rules and thresholds
- providing an easy interface to tune behavior

This approach keeps the logic transparent and easy to adjust without code changes.

## Core concept

Instead of toggling the charger on and off at fixed thresholds, this system maps solar power output to charging current in stepped increments. As solar production increases, charging current increases in a controlled way. When solar drops, the current reduces gracefully using time-based hysteresis.

This system uses these fixed stepped levels with time-based hysteresis:

- 0–1499 W solar → 0 A (no charging)
- 1500–1999 W solar → 6 A
- 2000–2499 W solar → 8 A
- 2500–2999 W solar → 10 A
- 3000–3499 W solar → 13 A
- 3500+ W solar → 15 A (maximum)

Hysteresis is time-based:
- Upgrade to next step after 3 minutes at higher threshold
- Downgrade to lower step after 2 minutes at lower threshold

This results in smoother, more stable charging behavior without rapid on/off cycling due to cloud cover or momentary fluctuations.

## Recommended Home Assistant entities

These are the typical sensors and controls required:

- `sensor.pv_power_watts` or `sensor.solar_generation_watts`
- `binary_sensor.ev_connected`
- `sensor.ev_charger_power_watts` (optional, for monitoring)
- `number.ev_charger_current_amps` (the control entity)
- `input_datetime.pv_last_step_change` (tracks when the last step change occurred)

## Home Assistant configuration

Create an input_datetime helper to track the last step change time:

```yaml
input_datetime:
  pv_last_step_change:
    name: "PV Last Step Change"
    has_date: true
    has_time: true
```

Optionally, add a helper to display the current step for debugging:

```yaml
input_text:
  pv_current_step:
    name: "Current PV Step"
    max: 20
    initial: "0"
```

## Safety override (recommended)

This optional helper lets you disable the PV automation without affecting other schedules or manual charging control.

```yaml
input_boolean:
  pv_charge_safety_override:
    name: "PV Charge Safety Override"
    initial: false
    icon: mdi:shield-check
```

When this switch is turned on, the PV automation will not change the charger current. Other schedules and manual control remain unaffected.

## Time window restriction (default: 10:05–15:55)

To allow charging only during a specific window, add the following helpers:

```yaml
input_datetime:
  pv_charge_window_start:
    name: "PV Charge Window Start"
    has_date: false
    has_time: true

  pv_charge_window_end:
    name: "PV Charge Window End"
    has_date: false
    has_time: true
```

Set the default values in Home Assistant to:
- PV Charge Window Start: 10:05
- PV Charge Window End: 15:55

Then add a template binary sensor:

```yaml
template:
  - binary_sensor:
      - name: PV Charge Window Active
        unique_id: pv_charge_window_active
        state: >
          {% set start = states('input_datetime.pv_charge_window_start') %}
          {% set end = states('input_datetime.pv_charge_window_end') %}
          {% set now_hm = now().strftime('%H:%M') %}
          {% if start in ['unknown', 'unavailable', 'none', ''] or end in ['unknown', 'unavailable', 'none', ''] %}
            {{ false }}
          {% elif start <= end %}
            {{ start <= now_hm and now_hm <= end }}
          {% else %}
            {{ now_hm >= start or now_hm <= end }}
          {% endif %}
```

Add this condition to your PV charge automation:

```yaml
condition:
  - condition: state
    entity_id: binary_sensor.ev_connected
    state: "on"
  - condition: state
    entity_id: binary_sensor.pv_charge_window_active
    state: "on"
  - condition: state
    entity_id: input_boolean.pv_charge_safety_override
    state: "off"
```

This means the automation will only run inside the window, and will not force a value outside the window. If the safety override is switched on, the PV automation becomes inactive without affecting other schedules or manual control.

## Example Home Assistant automation

This automation implements time-based hysteresis and respects the configured charging window and safety override. It evaluates whether a step change should occur based on:
1. Current solar power level
2. Time elapsed since last step change
3. Direction of change (up = 3 min delay, down = 2 min delay)
4. Whether the current time is inside the allowed charging window
5. Whether the safety override switch is off

```yaml
automation:
  - alias: "PV Charge Controller - Set charger current by solar output"
    trigger:
      - trigger: state
        entity_id: sensor.pv_power_watts
    condition:
      - condition: state
        entity_id: binary_sensor.ev_connected
        state: "on"
      - condition: state
        entity_id: binary_sensor.pv_charge_window_active
        state: "on"
      - condition: state
        entity_id: input_boolean.pv_charge_safety_override
        state: "off"
    action:
      - variables:
          pv_power: "{{ states('sensor.pv_power_watts') | float(0) }}"
          current_amps: "{{ states('number.ev_charger_current_amps') | float(0) }}"
          last_change: "{{ state_attr('input_datetime.pv_last_step_change', 'timestamp') | float(0) }}"
          time_since_change: "{{ (now().timestamp() - last_change) / 60 }}"

          target_step: >
            {%- if pv_power >= 3500 -%}
              6
            {%- elif pv_power >= 3000 -%}
              5
            {%- elif pv_power >= 2500 -%}
              4
            {%- elif pv_power >= 2000 -%}
              3
            {%- elif pv_power >= 1500 -%}
              2
            {%- else -%}
              1
            {%- endif %}

          target_amps: >
            {%- if pv_power >= 3500 -%}
              15
            {%- elif pv_power >= 3000 -%}
              13
            {%- elif pv_power >= 2500 -%}
              10
            {%- elif pv_power >= 2000 -%}
              8
            {%- elif pv_power >= 1500 -%}
              6
            {%- else -%}
              0
            {%- endif %}

          is_stepping_up: "{{ target_amps > current_amps }}"
          is_stepping_down: "{{ target_amps < current_amps }}"
          min_time_up: 3
          min_time_down: 2
          can_change_up: "{{ is_stepping_up and time_since_change >= min_time_up }}"
          can_change_down: "{{ is_stepping_down and time_since_change >= min_time_down }}"
          should_change: "{{ target_amps == current_amps or can_change_up or can_change_down }}"

      - if: "{{ should_change and target_amps != current_amps }}"
        then:
          - service: number.set_value
            target:
              entity_id: number.ev_charger_current_amps
            data:
              value: "{{ target_amps }}"

          - service: input_datetime.set_datetime
            target:
              entity_id: input_datetime.pv_last_step_change
            data:
              datetime: "{{ now().isoformat() }}"

          - service: input_text.set_value
            target:
              entity_id: input_text.pv_current_step
            data:
              value: "{{ target_step }}"
```

## How hysteresis works

Stepping UP (increasing charge current):
- Solar power rises above next threshold
- System waits 3 minutes to confirm sustained high production
- After 3 minutes, charge current increases to next step
- Prevents charging down when a cloud briefly passes

Stepping DOWN (decreasing charge current):
- Solar power falls below current threshold
- System waits 2 minutes to confirm sustained lower production
- After 2 minutes, charge current decreases to lower step
- Faster down-stepping allows quicker response to clouds while preventing flapping

## Step reference table

| Solar Power Range | Charge Current | Step | Up Delay | Down Delay |
|-------------------|----------------|------|----------|------------|
| 0–1499 W          | 0 A            | 1    | —        | 2 min      |
| 1500–1999 W       | 6 A            | 2    | 3 min    | 2 min      |
| 2000–2499 W       | 8 A            | 3    | 3 min    | 2 min      |
| 2500–2999 W       | 10 A           | 4    | 3 min    | 2 min      |
| 3000–3499 W       | 13 A           | 5    | 3 min    | 2 min      |
| 3500+ W           | 15 A           | 6    | 3 min    | —          |

## Dashboard integration

```yaml
type: entities
title: PV Charge Controller
entities:
  - entity: sensor.pv_power_watts
    name: Solar Power
  - entity: number.ev_charger_current_amps
    name: Charge Current
  - entity: input_text.pv_current_step
    name: Current Step
  - entity: binary_sensor.ev_connected
    name: EV Connected
  - entity: binary_sensor.pv_charge_window_active
    name: Charging Window Active
  - entity: input_boolean.pv_charge_safety_override
    name: Safety Override
  - entity: input_datetime.pv_last_step_change
    name: Last Step Change
```

## Recommended next steps

1. Identify your charger's current control entity in Home Assistant
2. Confirm your solar power sensor name
3. Create the input_datetime and input_text helpers
4. Create the charging window helpers with start = 10:05 and end = 15:55
5. Create the safety override helper
6. Deploy the automation with the time-based hysteresis logic
7. Test over a few sunny days to observe step change behavior
8. Adjust `min_time_up` and `min_time_down` if needed
9. Add the dashboard card for easy monitoring and debugging

## Notes

- Charging only occurs when `binary_sensor.ev_connected` is `on`
- Current is set to 0 A when solar is below 1500 W (disables charging)
- Time-based hysteresis prevents rapid step changes from cloud cover
- Stepping up takes 3 minutes; stepping down takes 2 minutes
- The optional time window restricts charging to 10:05–15:55 by default
- The automation does not force the charger to 0 A outside the window, so other scripts/schedules can still operate
- Turning on `input_boolean.pv_charge_safety_override` disables the PV automation entirely without affecting other schedules or manual control

---

This repository provides a practical, stepped approach to solar-aware EV charging in Home Assistant using fixed power thresholds with intelligent time-based hysteresis, optional time-window control, and a manual safety override.
