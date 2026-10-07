# Home Assistant PV Excess EV Charging

This project is designed for a Home Assistant setup where EV charging current is automatically adjusted based on available solar power generation. Rather than simple on/off control, this approach uses stepped charge current levels to match solar output, maximizing efficiency and grid independence.

## Why this fits Home Assistant

Home Assistant excels at:

- monitoring solar generation power (`sensor.pv_power_watts`)
- controlling EV charger current levels (`number.ev_charger_current_amps`)
- tracking EV connection state (`binary_sensor.ev_connected`)
- managing time-based rules and thresholds
- providing an easy interface to tune behavior

This approach keeps the logic transparent and easy to adjust without code changes.

## Core concept

Instead of toggling the charger on and off at fixed thresholds, this system maps solar power output to charging current in stepped increments. As solar production increases, charging current increases. As solar drops, charging current decreases proportionally.

This system uses these fixed stepped levels with **time-based hysteresis**:

- **0–1000 W solar** → 0 A (no charging)
- **1000–1500 W solar** → 6 A (minimum charge)
- **1500–2500 W solar** → 8 A
- **2500–3000 W solar** → 10 A
- **3000+ W solar** → 15 A (maximum)

**Hysteresis is time-based:**
- Upgrade to next step after **3 minutes** at higher threshold
- Downgrade to lower step after **2 minutes** at lower threshold

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

## Example Home Assistant automation

This automation implements time-based hysteresis. It evaluates whether a step change should occur based on:
1. Current solar power level
2. Time elapsed since last step change
3. Direction of change (up = 3 min delay, down = 2 min delay)

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
    action:
      - variables:
          pv_power: "{{ states('sensor.pv_power_watts') | float(0) }}"
          current_amps: "{{ states('number.ev_charger_current_amps') | float(0) }}"
          last_change: "{{ state_attr('input_datetime.pv_last_step_change', 'timestamp') | float(0) }}"
          time_since_change: "{{ (now().timestamp() - last_change) / 60 }}"
          
          target_step: >
            {%- if pv_power >= 3000 -%}
              5
            {%- elif pv_power >= 2500 -%}
              4
            {%- elif pv_power >= 1500 -%}
              3
            {%- elif pv_power >= 1000 -%}
              2
            {%- else -%}
              1
            {%- endif %}
          
          target_amps: >
            {%- if pv_power >= 3000 -%}
              15
            {%- elif pv_power >= 2500 -%}
              10
            {%- elif pv_power >= 1500 -%}
              8
            {%- elif pv_power >= 1000 -%}
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

**Stepping UP (increasing charge current):**
- Solar power rises above next threshold
- System waits **3 minutes** to confirm sustained high production
- After 3 minutes, charge current increases to next step
- Prevents charging down when a cloud briefly passes

**Stepping DOWN (decreasing charge current):**
- Solar power falls below current threshold
- System waits **2 minutes** to confirm sustained lower production
- After 2 minutes, charge current decreases to lower step
- Faster down-stepping allows quicker response to clouds while preventing flapping

## Step reference table

| Solar Power Range | Charge Current | Step | Up Delay | Down Delay |
|-------------------|----------------|------|----------|------------|
| 0–999 W           | 0 A            | 1    | —        | 2 min      |
| 1000–1499 W       | 6 A            | 2    | 3 min    | 2 min      |
| 1500–2499 W       | 8 A            | 3    | 3 min    | 2 min      |
| 2500–2999 W       | 10 A           | 4    | 3 min    | 2 min      |
| 3000+ W           | 15 A           | 5    | 3 min    | —          |

## Dashboard integration

Display a simple card in Home Assistant to monitor the system:

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
  - entity: sensor.ev_charger_power_watts
    name: Charging Power
  - entity: input_datetime.pv_last_step_change
    name: Last Step Change
```

Or use a more visual gauge-based card:

```yaml
type: vertical-stack
cards:
  - type: gauge
    entity: sensor.pv_power_watts
    min: 0
    max: 4000
    needle: true
    title: Solar Output
    
  - type: gauge
    entity: number.ev_charger_current_amps
    min: 0
    max: 32
    needle: true
    title: Charge Current
    
  - type: entities
    entities:
      - entity: binary_sensor.ev_connected
      - entity: input_text.pv_current_step
      - entity: input_datetime.pv_last_step_change
```

## Time-based hysteresis advantages

- **Prevents flapping**: Brief cloud cover won't trigger rapid step changes
- **Asymmetric response**: Favors charging up (3 min delay) while responding quickly to clouds (2 min delay)
- **Predictable behavior**: Users know exactly how long a solar level must be sustained
- **Natural**: Mimics real-world solar behavior patterns
- **Easy tuning**: Adjust `min_time_up` and `min_time_down` values to taste

## Customizing the timing

To adjust hysteresis timing, modify these values in the automation:

```yaml
min_time_up: 3        # Minutes to wait before stepping up
min_time_down: 2      # Minutes to wait before stepping down
```

Example for faster response:
```yaml
min_time_up: 1        # Step up quickly
min_time_down: 1      # Step down quickly
```

Example for more conservative operation:
```yaml
min_time_up: 5        # Longer confirmation period
min_time_down: 3      # Slower down-stepping
```

## Example project layout

```text
.
├── README.md
├── docs/
│   ├── stepped_charging_strategy.md
│   ├── hysteresis_explanation.md
│   └── tuning_guide.md
├── examples/
│   ├── homeassistant.yaml
│   └── fixed_step_configuration.yaml
└── scripts/
    └── pv_charge_controller.py
```

## How it works

1. **EV must be connected** - Automation only runs when `binary_sensor.ev_connected` is `on`
2. **Solar power is read** - Real-time power from `sensor.pv_power_watts`
3. **Target step is calculated** - Power is compared against thresholds
4. **Time is checked** - How long since the last step change is evaluated
5. **Hysteresis is applied** - Up steps require 3 min, down steps require 2 min
6. **Charge current is set** - If conditions are met, `number.ev_charger_current_amps` is updated
7. **Timestamp is recorded** - Last change time is saved for next evaluation

The system responds thoughtfully to solar fluctuations with built-in protection against rapid switching.

## Advantages of time-based hysteresis

- **Smooth operation**: Stepped current avoids on/off switching
- **Grid-friendly**: Steady charging reduces power quality issues
- **Efficient**: Uses available solar without oversizing
- **Simple configuration**: Fixed steps with fixed timing delays
- **Protective**: Prevents rapid changes from brief cloud cover
- **Responsive**: Asymmetric timing allows quick reduction when needed
- **Transparent**: Clear, predictable behavior

## Recommended next steps

1. Identify your charger's current control entity in Home Assistant
2. Confirm your solar power sensor name
3. Create the input_datetime and input_text helpers
4. Deploy the automation with the time-based hysteresis logic
5. Test over a few sunny days to observe step change behavior
6. Adjust `min_time_up` and `min_time_down` if needed
7. Add the dashboard card for easy monitoring and debugging

## Notes

- Charging only occurs when `binary_sensor.ev_connected` is `on`
- Current is set to 0 A when solar is below 1000 W (disables charging)
- Time-based hysteresis prevents rapid step changes from cloud cover
- Stepping up takes 3 minutes; stepping down takes 2 minutes
- Last change timestamp is logged for visibility and debugging

---

This repository provides a practical, stepped approach to solar-aware EV charging in Home Assistant using fixed power thresholds with intelligent time-based hysteresis.
