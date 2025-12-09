# Trigger-based template binary sensor frontend API

This document describes the backend API for frontend engineers implementing the UI for trigger-based template binary sensors.

## Overview

Trigger-based template binary sensors allow users to create binary sensors that update based on events (triggers) rather than continuously watching state changes. This is similar to how trigger-based template sensors work in YAML configurations.

## Config flow structure

### Menu hierarchy

When a user selects "Binary Sensor" from the initial template type menu, they now see a sub-menu:

```
Template Type Menu
└── Binary Sensor
    ├── State-based (existing functionality)
    └── Trigger-based (new functionality)
```

### Flow steps

1. **Initial menu** (`async_step_user`): User selects "binary_sensor"
2. **Binary sensor sub-menu** (`async_step_binary_sensor`): User chooses between:
   - `binary_sensor_state_based`: Traditional state-based binary sensor
   - `binary_sensor_trigger_based`: New trigger-based binary sensor
3. **Configuration form**: Based on selection, shows appropriate form

## API schemas

### Trigger-based binary sensor form schema

The trigger-based binary sensor configuration uses these selectors:

```python
TRIGGER_BINARY_SENSOR_SCHEMA = {
    # Required fields
    vol.Required(CONF_NAME): TextSelector(),
    vol.Required(CONF_STATE): TemplateSelector(),
    vol.Required(CONF_TRIGGERS): TriggerSelector(),

    # Optional fields (in advanced options section)
    vol.Optional(CONF_DEVICE_CLASS): SelectSelector(
        SelectSelectorConfig(
            options=[cls.value for cls in BinarySensorDeviceClass],
            mode=SelectSelectorMode.DROPDOWN,
            translation_key="binary_sensor_device_class",
            sort=True,
        )
    ),
    vol.Optional(CONF_DELAY_ON): DurationSelector(),
    vol.Optional(CONF_DELAY_OFF): DurationSelector(),
    vol.Optional(CONF_AUTO_OFF): DurationSelector(),
    vol.Optional(CONF_CONDITIONS): ConditionSelector(),
    vol.Optional(CONF_ACTIONS): ActionSelector(),
    vol.Optional(CONF_VARIABLES): ObjectSelector(),
}
```

### Selectors used

| Field | Selector | Description |
|-------|----------|-------------|
| `name` | `TextSelector` | Name for the binary sensor entity |
| `state` | `TemplateSelector` | Template that evaluates to true/false |
| `triggers` | `TriggerSelector` | One or more triggers (event, state, time, etc.) |
| `device_class` | `SelectSelector` | Optional device class (motion, door, etc.) |
| `delay_on` | `DurationSelector` | Delay before turning on |
| `delay_off` | `DurationSelector` | Delay before turning off |
| `auto_off` | `DurationSelector` | Auto turn off after duration |
| `conditions` | `ConditionSelector` | Conditions that must be met for trigger |
| `actions` | `ActionSelector` | Actions to run when triggered |
| `variables` | `ObjectSelector` | Custom variables for templates |

## Config entry structure

### Options storage

Trigger-based binary sensors store their configuration in `config_entry.options`:

```python
{
    "template_type": "binary_sensor",
    "trigger_based": True,  # Flag distinguishing from state-based
    "name": "Motion Detected",
    "state": "{{ trigger.event.data.detected }}",
    "triggers": [
        {
            "trigger": "event",
            "event_type": "motion_detected"
        }
    ],
    # Optional advanced options
    "advanced_options": {
        "device_class": "motion",
        "auto_off": {"hours": 0, "minutes": 5, "seconds": 0},
        "conditions": [...],
        "actions": [...],
        "variables": {...}
    }
}
```

### Key fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_type` | string | Yes | Always "binary_sensor" |
| `trigger_based` | boolean | Yes | `true` for trigger-based, absent for state-based |
| `name` | string | Yes | Entity name |
| `state` | string | Yes | Jinja2 template for state |
| `triggers` | list | Yes | List of trigger configurations |
| `advanced_options` | object | No | Container for optional advanced settings |

## Trigger context

When a trigger fires, the following variables are available in templates:

- `trigger`: Full trigger context object
- `trigger.event`: For event triggers, contains the event data
- `trigger.entity_id`: For state triggers, the entity that triggered
- `trigger.to_state`: For state triggers, the new state
- `trigger.from_state`: For state triggers, the previous state

Example state template:
```jinja2
{{ trigger.event.data.motion == true }}
```

## Preview support

The frontend can request a preview of the binary sensor state using the existing WebSocket API:

```json
{
    "type": "template/start_preview",
    "flow_id": "<flow_id>",
    "flow_type": "config_flow",
    "user_input": {
        "name": "Motion Sensor",
        "state": "{{ trigger.event.data.detected }}",
        "triggers": [{"trigger": "event", "event_type": "motion_event"}]
    }
}
```

**Note**: For trigger-based sensors, the preview will show "unknown" state since no triggers have fired yet during preview.

## Differences from state-based binary sensors

| Aspect | State-based | Trigger-based |
|--------|-------------|---------------|
| Updates | Continuous state watching | Event-driven |
| Template context | `states`, `is_state`, etc. | `trigger` object |
| Initial state | Computed immediately | Unknown until first trigger |
| Use case | Monitor entity states | React to events |

## Migration considerations

### Existing config entries

Existing binary sensor config entries (without `trigger_based` flag) continue to work as state-based sensors. No migration is needed.

### YAML to UI migration

Users migrating from YAML trigger-based binary sensors should:
1. Select "Binary Sensor" → "Trigger-based"
2. Copy their trigger configuration
3. Copy their state template
4. Configure any advanced options

## Error handling

### Invalid triggers

If the trigger configuration is invalid, the config flow will show an error. Common issues:
- Missing required trigger fields
- Invalid event_type values
- Malformed condition syntax

### Template errors

Template validation occurs during config flow. Errors are displayed inline in the form.

## Testing the implementation

To test the trigger-based binary sensor via API:

```python
# Create a trigger-based binary sensor config entry
config_entry = MockConfigEntry(
    domain="template",
    options={
        "name": "Test Trigger Sensor",
        "template_type": "binary_sensor",
        "trigger_based": True,
        "triggers": [{"trigger": "event", "event_type": "test_event"}],
        "state": "{{ trigger.event.data.value }}",
    },
)
config_entry.add_to_hass(hass)
await hass.config_entries.async_setup(config_entry.entry_id)

# Fire the trigger
hass.bus.async_fire("test_event", {"value": True})
await hass.async_block_till_done()

# Check the state
state = hass.states.get("binary_sensor.test_trigger_sensor")
assert state.state == "on"
```

## WebSocket API endpoints

The standard template integration WebSocket endpoints work with trigger-based sensors:

- `template/start_preview`: Start live preview during config flow
- Config entries CRUD via standard `config_entries/*` endpoints

## Localization

New translation keys in `strings.json`:

```json
{
  "config": {
    "step": {
      "binary_sensor": {
        "menu_options": {
          "binary_sensor_state_based": "State-based binary sensor",
          "binary_sensor_trigger_based": "Trigger-based binary sensor"
        }
      },
      "binary_sensor_trigger_based": {
        "description": "A trigger-based template binary sensor updates its state when specific triggers fire, such as events or state changes.",
        "data": {
          "name": "Name",
          "state": "State template",
          "triggers": "Triggers"
        },
        "data_description": {
          "state": "Template that returns true or false. Use {{ trigger }} to access trigger data.",
          "triggers": "Events or conditions that update the sensor state"
        }
      }
    }
  }
}
```
