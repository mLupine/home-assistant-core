# Trigger-Based Template Binary Sensor UI Configuration - Implementation Plan

## Executive Summary

This document outlines the implementation plan for adding UI (config flow) support for trigger-based template binary sensors in Home Assistant. Currently, trigger-based template entities can only be configured via YAML; this feature will enable users to configure them through the frontend UI.

## Current Architecture Analysis

### Existing Components

1. **Config Flow** (`config_flow.py`)
   - Uses `SchemaConfigFlowHandler` for schema-based config flows
   - Menu-driven selection of template type (binary_sensor, sensor, etc.)
   - State-based templates only - no trigger support
   - Stores configuration in `config_entry.options`
   - Supports preview via WebSocket API

2. **Binary Sensor Platform** (`binary_sensor.py`)
   - `StateBinarySensorEntity`: State-tracking template entity (UI-configured)
   - `TriggerBinarySensorEntity`: Trigger-based entity (YAML-only)
   - `async_setup_entry`: Only creates `StateBinarySensorEntity`
   - `async_setup_platform`: Handles both state and trigger entities from YAML

3. **Coordinator** (`coordinator.py`)
   - `TriggerUpdateCoordinator`: Manages trigger subscriptions and entity updates
   - Created/destroyed in `_process_config()` during YAML reload
   - No lifecycle management for config entries

4. **Trigger Entity Base** (`trigger_entity.py`)
   - `TriggerEntity`: Base class for trigger-based template entities
   - Requires a `TriggerUpdateCoordinator` instance
   - Handles trigger variables and template rendering

5. **Selectors** (`homeassistant/helpers/selector.py`)
   - `TriggerSelector`: Already exists, validates trigger configurations
   - `ConditionSelector`: Already exists for conditions
   - `ActionSelector`: Already exists for actions

### Data Flow

```
YAML Config:
template:
  - triggers: [...]        ──┐
    binary_sensor:           │
      - state: "..."         │
                             ▼
              ┌──────────────────────────┐
              │  _process_config()       │
              │  Creates coordinator     │
              │  Discovers entities      │
              └──────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────┐
              │  async_setup_platform()  │
              │  TriggerBinarySensorEntity│
              └──────────────────────────┘

Config Entry (Current - State Only):
config_entry.options = {
  "template_type": "binary_sensor",
  "state": "{{ ... }}",
  ...
}
                             │
                             ▼
              ┌──────────────────────────┐
              │  async_setup_entry()     │
              │  StateBinarySensorEntity │
              └──────────────────────────┘
```

## Implementation Plan

### Phase 1: Config Flow Changes

#### 1.1 Add Sub-Menu for Binary Sensor Type Selection

Modify `config_flow.py` to add a sub-menu when binary_sensor is selected:

```python
# New menu options for binary sensor
BINARY_SENSOR_TYPES = ["state_based", "trigger_based"]

# New flow structure
CONFIG_FLOW = {
    "user": SchemaFlowMenuStep(TEMPLATE_TYPES, True),
    # ... existing steps ...
    Platform.BINARY_SENSOR: SchemaFlowMenuStep(BINARY_SENSOR_TYPES),
    "binary_sensor_state_based": SchemaFlowFormStep(...),
    "binary_sensor_trigger_based": SchemaFlowFormStep(...),
}
```

#### 1.2 Create Trigger-Based Binary Sensor Schema

```python
def generate_trigger_binary_sensor_schema(flow_type: str) -> vol.Schema:
    """Generate schema for trigger-based binary sensor."""
    schema: dict[vol.Marker, Any] = {}

    if flow_type == "config":
        schema[vol.Required(CONF_NAME)] = selector.TextSelector()

    schema.update({
        vol.Required(CONF_TRIGGERS): selector.TriggerSelector(),
        vol.Optional(CONF_CONDITIONS): selector.ConditionSelector(),
        vol.Optional(CONF_ACTIONS): selector.ActionSelector(),
        vol.Optional(CONF_VARIABLES): selector.ObjectSelector(),
        vol.Required(CONF_STATE): selector.TemplateSelector(),
        vol.Optional(CONF_AUTO_OFF): selector.DurationSelector(),
        vol.Optional(CONF_DELAY_ON): selector.DurationSelector(),
        vol.Optional(CONF_DELAY_OFF): selector.DurationSelector(),
        vol.Optional(CONF_DEVICE_CLASS): selector.SelectSelector(...),
        vol.Optional(CONF_DEVICE_ID): selector.DeviceSelector(),
        vol.Optional(CONF_ADVANCED_OPTIONS): section(...),
    })

    return vol.Schema(schema)
```

#### 1.3 Add Trigger Configuration Validation

```python
async def validate_trigger_user_input(
    handler: SchemaCommonFlowHandler,
    user_input: dict[str, Any],
) -> dict[str, Any]:
    """Validate and process trigger configuration."""
    # Validate triggers
    hass = handler.hass
    triggers = await async_validate_trigger_config(hass, user_input[CONF_TRIGGERS])

    # Validate conditions if present
    if CONF_CONDITIONS in user_input:
        conditions = await async_validate_conditions_config(
            hass, user_input[CONF_CONDITIONS]
        )
        user_input[CONF_CONDITIONS] = conditions

    user_input[CONF_TRIGGERS] = triggers
    return {"template_type": "binary_sensor", "trigger_based": True} | user_input
```

#### 1.4 Update Config Entry Version

Bump `MINOR_VERSION` to handle new trigger-based format:

```python
class TemplateConfigFlowHandler(SchemaConfigFlowHandler, domain=DOMAIN):
    MINOR_VERSION = 3  # Bumped for trigger support
    VERSION = 1
```

### Phase 2: Entity Setup Changes

#### 2.1 Modify `async_setup_entry` in `binary_sensor.py`

```python
async def async_setup_entry(
    hass: HomeAssistant,
    config_entry: ConfigEntry,
    async_add_entities: AddConfigEntryEntitiesCallback,
) -> None:
    """Initialize config entry."""
    options = config_entry.options

    if options.get("trigger_based"):
        # Trigger-based setup
        await async_setup_trigger_entry(
            hass,
            config_entry,
            async_add_entities,
        )
    else:
        # State-based setup (existing behavior)
        await async_setup_template_entry(
            hass,
            config_entry,
            async_add_entities,
            StateBinarySensorEntity,
            BINARY_SENSOR_CONFIG_ENTRY_SCHEMA,
        )
```

#### 2.2 Create `async_setup_trigger_entry` Helper

New function in `helpers.py`:

```python
async def async_setup_trigger_entry(
    hass: HomeAssistant,
    config_entry: ConfigEntry,
    async_add_entities: AddConfigEntryEntitiesCallback,
    trigger_entity_cls: type[TriggerEntity],
    config_schema: vol.Schema,
) -> None:
    """Set up trigger-based template entity from config entry."""
    options = dict(config_entry.options)
    options.pop("template_type")
    options.pop("trigger_based", None)

    if advanced_options := options.pop(CONF_ADVANCED_OPTIONS, None):
        options = {**options, **advanced_options}

    # Create coordinator
    coordinator = TriggerUpdateCoordinator(hass, options)

    # Store coordinator reference for cleanup
    config_entry.runtime_data = coordinator

    # Setup coordinator (attach triggers)
    await coordinator.async_setup({})

    # Create entity
    entity = trigger_entity_cls(hass, coordinator, options)
    async_add_entities([entity])
```

### Phase 3: Coordinator Lifecycle Management

#### 3.1 Modify `__init__.py` for Config Entry Coordinator Management

```python
async def async_setup_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    """Set up a config entry."""
    # Existing validation...

    await hass.config_entries.async_forward_entry_setups(
        entry, (entry.options["template_type"],)
    )

    # Listen for labs feature flag changes
    entry.async_on_unload(
        async_labs_listen(
            hass,
            AUTOMATION_DOMAIN,
            NEW_TRIGGERS_CONDITIONS_FEATURE_FLAG,
            partial(hass.config_entries.async_schedule_reload, entry.entry_id),
        )
    )

    return True


async def async_unload_entry(hass: HomeAssistant, entry: ConfigEntry) -> bool:
    """Unload a config entry."""
    # Clean up coordinator if this is a trigger-based entry
    if hasattr(entry, 'runtime_data') and entry.runtime_data:
        coordinator = entry.runtime_data
        coordinator.async_remove()

    return await hass.config_entries.async_unload_platforms(
        entry, (entry.options["template_type"],)
    )
```

#### 3.2 Coordinator Entry Support

Modify `TriggerUpdateCoordinator` to support config entries:

```python
class TriggerUpdateCoordinator(DataUpdateCoordinator):
    def __init__(
        self,
        hass: HomeAssistant,
        config: ConfigType,
        config_entry: ConfigEntry | None = None,
    ) -> None:
        """Instantiate trigger data."""
        super().__init__(
            hass,
            _LOGGER,
            config_entry=config_entry,  # Pass for proper lifecycle
            name="Trigger Update Coordinator"
        )
        # ... rest of init
```

### Phase 4: Strings and UI Support

#### 4.1 Update `strings.json`

Add new entries for trigger-based binary sensor configuration:

```json
{
  "config": {
    "step": {
      "binary_sensor": {
        "menu_options": {
          "binary_sensor_state_based": "State-based binary sensor",
          "binary_sensor_trigger_based": "Trigger-based binary sensor"
        },
        "description": "Choose the type of binary sensor template to create."
      },
      "binary_sensor_state_based": {
        // ... existing state-based fields
      },
      "binary_sensor_trigger_based": {
        "data": {
          "name": "[%key:common::config_flow::data::name%]",
          "triggers": "Triggers",
          "conditions": "Conditions",
          "actions": "Actions",
          "variables": "Variables",
          "state": "[%key:component::template::common::state%]",
          "auto_off": "Auto off",
          "delay_on": "Delay on",
          "delay_off": "Delay off",
          "device_class": "[%key:component::template::common::device_class%]",
          "device_id": "[%key:common::config_flow::data::device%]"
        },
        "data_description": {
          "triggers": "Define triggers that will update the binary sensor state. The sensor will only update when a trigger fires.",
          "conditions": "Optional conditions that must be met after a trigger fires before the sensor updates.",
          "actions": "Optional actions to execute when a trigger fires, before updating the sensor state.",
          "variables": "Variables available to templates. Trigger variables are always available.",
          "state": "Template that evaluates to the binary sensor state. Has access to trigger variables.",
          "auto_off": "Automatically turn off the binary sensor after this duration. Requires a trigger.",
          "delay_on": "Delay before turning on.",
          "delay_off": "Delay before turning off.",
          "device_id": "[%key:component::template::common::device_id_description%]"
        },
        "title": "Trigger-based template binary sensor"
      }
    }
  }
}
```

### Phase 5: Preview Support

#### 5.1 Create Preview Entity for Trigger-Based Binary Sensors

Preview for trigger-based entities is more complex since they don't update continuously. Options:

1. **Simulate trigger firing**: Create a mock trigger event for preview
2. **Show static preview**: Display the template with mock trigger variables
3. **Disable preview**: Return a message explaining trigger-based entities can't be previewed

**Recommended approach**: Show static preview with mock trigger data.

```python
@callback
def async_create_preview_trigger_binary_sensor(
    hass: HomeAssistant, name: str, config: dict[str, Any]
) -> TriggerBinarySensorEntity:
    """Create a preview trigger binary sensor."""
    # Create a mock coordinator for preview
    mock_coordinator = MockPreviewCoordinator(hass)
    mock_coordinator.data = {
        "run_variables": {"trigger": {"platform": "preview"}},
        "context": None,
    }

    return TriggerBinarySensorEntity(hass, mock_coordinator, config | {CONF_NAME: name})
```

### Phase 6: Testing Strategy

#### 6.1 Unit Tests for Config Flow

```python
# tests/components/template/test_config_flow.py

@pytest.mark.parametrize(
    ("trigger_config", "expected_state"),
    [
        (
            {
                "triggers": [{"platform": "state", "entity_id": "input_boolean.test"}],
                "state": "{{ trigger.to_state.state == 'on' }}",
            },
            "off",  # Initial state before trigger fires
        ),
    ],
)
async def test_trigger_binary_sensor_config_flow(
    hass: HomeAssistant,
    trigger_config: dict,
    expected_state: str,
) -> None:
    """Test trigger-based binary sensor config flow."""
    result = await hass.config_entries.flow.async_init(
        DOMAIN, context={"source": config_entries.SOURCE_USER}
    )
    assert result["type"] is FlowResultType.MENU

    result = await hass.config_entries.flow.async_configure(
        result["flow_id"],
        {"next_step_id": "binary_sensor"},
    )
    assert result["type"] is FlowResultType.MENU

    result = await hass.config_entries.flow.async_configure(
        result["flow_id"],
        {"next_step_id": "binary_sensor_trigger_based"},
    )
    assert result["type"] is FlowResultType.FORM

    result = await hass.config_entries.flow.async_configure(
        result["flow_id"],
        {"name": "Test Trigger Sensor", **trigger_config},
    )
    assert result["type"] is FlowResultType.CREATE_ENTRY
```

#### 6.2 Integration Tests for Trigger Functionality

```python
# tests/components/template/test_binary_sensor.py

async def test_trigger_binary_sensor_from_config_entry(hass: HomeAssistant) -> None:
    """Test trigger-based binary sensor created from config entry."""
    config_entry = MockConfigEntry(
        domain=DOMAIN,
        data={},
        options={
            "name": "Test Trigger Sensor",
            "template_type": "binary_sensor",
            "trigger_based": True,
            "triggers": [{"platform": "state", "entity_id": "input_boolean.test"}],
            "state": "{{ trigger.to_state.state == 'on' }}",
        },
    )
    config_entry.add_to_hass(hass)

    hass.states.async_set("input_boolean.test", "off")

    await hass.config_entries.async_setup(config_entry.entry_id)
    await hass.async_block_till_done()

    # Verify entity created
    state = hass.states.get("binary_sensor.test_trigger_sensor")
    assert state is not None

    # Fire trigger
    hass.states.async_set("input_boolean.test", "on")
    await hass.async_block_till_done()

    state = hass.states.get("binary_sensor.test_trigger_sensor")
    assert state.state == "on"
```

#### 6.3 Test Coordinator Lifecycle

```python
async def test_trigger_binary_sensor_unload(hass: HomeAssistant) -> None:
    """Test trigger-based binary sensor coordinator cleanup on unload."""
    config_entry = MockConfigEntry(
        domain=DOMAIN,
        data={},
        options={
            "name": "Test Trigger Sensor",
            "template_type": "binary_sensor",
            "trigger_based": True,
            "triggers": [{"platform": "time_pattern", "minutes": "/5"}],
            "state": "{{ true }}",
        },
    )
    config_entry.add_to_hass(hass)

    await hass.config_entries.async_setup(config_entry.entry_id)
    await hass.async_block_till_done()

    # Verify coordinator is stored
    assert config_entry.runtime_data is not None
    coordinator = config_entry.runtime_data

    # Unload entry
    await hass.config_entries.async_unload(config_entry.entry_id)
    await hass.async_block_till_done()

    # Verify cleanup (trigger unsubscribed)
    assert coordinator._unsub_trigger is None
```

#### 6.4 Backward Compatibility Tests

```python
async def test_state_based_binary_sensor_still_works(hass: HomeAssistant) -> None:
    """Test existing state-based binary sensor config entries still work."""
    # Existing config entry format (no trigger_based key)
    config_entry = MockConfigEntry(
        domain=DOMAIN,
        data={},
        options={
            "name": "Test State Sensor",
            "template_type": "binary_sensor",
            "state": "{{ states('input_boolean.test') == 'on' }}",
        },
    )
    config_entry.add_to_hass(hass)

    hass.states.async_set("input_boolean.test", "on")

    await hass.config_entries.async_setup(config_entry.entry_id)
    await hass.async_block_till_done()

    state = hass.states.get("binary_sensor.test_state_sensor")
    assert state.state == "on"
```

### Phase 7: Migration Considerations

#### 7.1 Config Entry Migration

No migration needed for existing entries - they don't have `trigger_based` key, so they continue to work as state-based sensors.

#### 7.2 YAML Compatibility

YAML configuration remains unchanged and fully supported.

## File Changes Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `config_flow.py` | Modify | Add trigger-based binary sensor flow steps, schema, validation |
| `binary_sensor.py` | Modify | Update `async_setup_entry` to handle trigger-based config |
| `helpers.py` | Modify | Add `async_setup_trigger_entry` helper function |
| `__init__.py` | Modify | Update `async_unload_entry` for coordinator cleanup |
| `coordinator.py` | Modify | Add config_entry support to constructor |
| `const.py` | Modify | Add new constants if needed |
| `strings.json` | Modify | Add UI strings for trigger configuration |
| `test_config_flow.py` | Modify | Add tests for trigger-based config flow |
| `test_binary_sensor.py` | Modify | Add tests for trigger-based entity functionality |

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Coordinator lifecycle issues | Store coordinator in `runtime_data`, ensure cleanup in `async_unload_entry` |
| Breaking existing config entries | New `trigger_based` flag distinguishes new entries, existing entries unchanged |
| Complex trigger validation | Reuse existing `async_validate_trigger_config` from automation helpers |
| Preview complexity for triggers | Provide static preview with mock trigger variables |
| UI complexity | Use sub-menu to separate state-based and trigger-based flows |

## Implementation Milestones

- [x] **Milestone 1**: Config flow changes (sub-menu, schema, validation)
- [ ] **Milestone 2**: Entity setup changes (`async_setup_entry` modification)
- [ ] **Milestone 3**: Coordinator lifecycle management
- [x] **Milestone 4**: Strings and UI support
- [x] **Milestone 5**: Preview support for trigger entities (static preview implemented)
- [ ] **Milestone 6**: Comprehensive test coverage
- [ ] **Milestone 7**: Documentation updates

## Progress Tracking

| Milestone | Status | Tests Passing | Notes |
|-----------|--------|---------------|-------|
| 1 | Complete | 78/78 | Config flow sub-menu and schema implemented |
| 2 | In Progress | N/A | Need to implement entity setup for trigger-based entries |
| 3 | Pending | N/A | |
| 4 | Complete | 167/167 | Strings added and translations generated |
| 5 | Complete | N/A | Static preview using state-based entity fallback |
| 6 | Pending | N/A | Need tests for trigger-based config entries |
| 7 | Pending | N/A | |

## Completed Changes

### Phase 1 (Complete)
- Added `CONF_TRIGGER_BASED` constant to `const.py`
- Added `generate_trigger_binary_sensor_schema()` function in `config_flow.py`
- Added `validate_trigger_binary_sensor_user_input()` validation function
- Added `BINARY_SENSOR_TYPES` sub-menu options
- Modified `CONFIG_FLOW` to use sub-menu for binary_sensor
- Added `binary_sensor_state_based` and `binary_sensor_trigger_based` form steps
- Modified `OPTIONS_FLOW` to handle trigger-based options
- Updated `choose_options_step()` to route trigger-based entries correctly
- Added preview entity creator for trigger-based binary sensors
- Bumped `MINOR_VERSION` to 3

### Phase 4 (Complete)
- Updated `strings.json` with:
  - Binary sensor sub-menu text
  - State-based binary sensor step
  - Trigger-based binary sensor step
  - Options flow entries for both types
- Ran translation script to expand key references

### Phase 5 (Complete)
- Added `async_create_preview_trigger_binary_sensor()` function
- Uses static preview (state-based preview fallback) since triggers can't fire in preview mode

### Test Updates
- Added `_navigate_to_template_form()` helper in `test_config_flow.py`
- Updated all 8 occurrences of binary_sensor navigation in tests
- Updated `async_get_flow_preview_state()` in `conftest.py`
- All 78 config_flow tests pass
- All 167 binary_sensor tests pass

---

*Last Updated: Phase 1 & 4 Complete*
