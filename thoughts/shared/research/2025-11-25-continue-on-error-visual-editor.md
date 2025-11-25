---
date: 2025-11-25T02:58:11Z
researcher: Claude
git_commit: e7650638a80f9f8f4409c9f400eb0287bd620714
branch: dev
repository: core
topic: "Continue on Error Toggle in Visual Editor"
tags: [research, codebase, automation, actions, frontend-integration, continue_on_error]
status: complete
last_updated: 2025-11-25
last_updated_by: Claude
---

# Research: Continue on Error Toggle in Visual Editor

**Date**: 2025-11-25T02:58:11Z
**Researcher**: Claude
**Git Commit**: e7650638a80f9f8f4409c9f400eb0287bd620714
**Branch**: dev
**Repository**: core (Home Assistant Core)

## Research Question

How is `continue_on_error` implemented in the Home Assistant backend, and what does the frontend need to do to expose this option in the Visual Automation Editor?

## Summary

The `continue_on_error` property is **fully implemented in the backend** and ready for frontend integration. It's defined as an optional boolean property in the base action schema that all action types inherit. The frontend simply needs to:

1. Add a toggle/checkbox to the action card UI
2. Include `continue_on_error: true/false` in the action JSON when saving
3. The backend will validate, store, and execute it automatically

No backend changes are required. The property works identically to `enabled` and `alias` which are already exposed in the UI.

## Detailed Findings

### Backend Implementation

#### 1. Constant Definition
**File**: `homeassistant/const.py:101`
```python
CONF_CONTINUE_ON_ERROR: Final = "continue_on_error"
```

#### 2. Schema Definition (All Actions Inherit This)
**File**: `homeassistant/helpers/config_validation.py:1463-1467`
```python
SCRIPT_ACTION_BASE_SCHEMA: VolDictType = {
    vol.Optional(CONF_ALIAS): string,
    vol.Optional(CONF_CONTINUE_ON_ERROR): boolean,
    vol.Optional(CONF_ENABLED): vol.Any(boolean, template),
}
```

This base schema is spread into ALL 16 action type schemas:
- Scene activation
- Service/action calls
- Conditions
- Choose/if/then/else
- Delays
- Device automations
- Event firing
- Parallel execution
- Repeat loops
- Sequences
- Stop actions
- Variables
- Wait for trigger
- Wait template
- Set conversation response

#### 3. Execution Logic
**File**: `homeassistant/helpers/script.py:491-579`

The property is read at the start of each action step:
```python
async def _async_step(self, log_exceptions: bool) -> None:
    continue_on_error = self._action.get(CONF_CONTINUE_ON_ERROR, False)
    # ... action execution ...
    try:
        await getattr(self, handler)()
    except Exception as ex:
        self._handle_exception(ex, continue_on_error, ...)
```

#### 4. Error Handling Behavior
**File**: `homeassistant/helpers/script.py:549-579`

When `continue_on_error: true`:
- Errors are logged but execution continues to next action
- Only `HomeAssistantError` subclasses can be suppressed
- These exceptions ALWAYS stop regardless of the flag:
  - `_StopScript` (explicit stop action)
  - `vol.Invalid` (configuration errors)
  - `TemplateError` (template syntax errors)
  - `ServiceNotFound` (service doesn't exist)
  - `InvalidEntityFormatError` (bad entity ID)
  - `NoEntitySpecifiedError` (missing entity)
  - `ConditionError` (condition evaluation errors)
  - Non-HomeAssistant exceptions (library errors)

### API Data Flow

#### HTTP REST API
- **Endpoint**: `POST /api/config/automation/config/{config_key}`
- **File**: `homeassistant/components/config/automation.py:44-53`

Frontend sends JSON with actions array. Each action can include `continue_on_error`:
```json
{
  "alias": "My Automation",
  "triggers": [...],
  "actions": [
    {
      "action": "light.turn_on",
      "target": {"entity_id": "light.bedroom"},
      "continue_on_error": true
    },
    {
      "action": "notify.mobile_app",
      "data": {"message": "Light turned on"}
    }
  ]
}
```

#### WebSocket API
- **Command**: `automation/config`
- **File**: `homeassistant/components/automation/__init__.py:1175-1195`
- Returns `raw_config` which includes all action properties including `continue_on_error`

### Similar Properties Already in UI

The `enabled` and `alias` properties follow the exact same pattern and are already exposed in the Visual Editor:

| Property | Type | Default | Template Support | UI Status |
|----------|------|---------|------------------|-----------|
| `alias` | string | varies | No | **In UI** |
| `enabled` | boolean | True | Yes | **In UI** |
| `continue_on_error` | boolean | False | No | **Not in UI** |

### Frontend Metadata Pattern

**File**: `homeassistant/helpers/config_validation.py:1519,1697,1919,2015`

The frontend can store UI-specific data in a `metadata` field that core removes during validation:
```python
vol.Remove("metadata"): dict,  # Frontend can store UI state here
```

This is NOT where `continue_on_error` should go - it should be a direct property on the action object since the backend processes it.

## Code References

### Core Files
- `homeassistant/const.py:101` - Constant definition
- `homeassistant/helpers/config_validation.py:1463-1467` - Schema definition
- `homeassistant/helpers/script.py:491-530` - Step execution
- `homeassistant/helpers/script.py:549-579` - Error handling logic

### API Files
- `homeassistant/components/config/automation.py:44-53` - HTTP API view
- `homeassistant/components/config/view.py:101-141` - POST handler
- `homeassistant/components/automation/__init__.py:1175-1195` - WebSocket handler

### Test Files
- `tests/helpers/test_script.py:5992-6058` - Basic functionality test
- `tests/helpers/test_script.py:6061-6117` - Tests for exceptions that cannot be suppressed

## Architecture Documentation

### Property Access Pattern
```python
# Standard pattern used throughout script.py
continue_on_error = self._action.get(CONF_CONTINUE_ON_ERROR, False)
```

### Data Flow
```
Frontend (Visual Editor)
    ↓
JSON POST to /api/config/automation/config/{id}
    ↓
Schema Validation (SCRIPT_ACTION_BASE_SCHEMA validates continue_on_error as boolean)
    ↓
Write to automations.yaml
    ↓
Reload automation
    ↓
Script Execution (reads continue_on_error per action)
    ↓
Error Handling (_handle_exception uses the flag)
```

### Frontend Integration Requirements

1. **Add Toggle Component**: In the action row component, add a checkbox/toggle for "Continue on Error"

2. **Bind to Property**: The toggle should bind to `action.continue_on_error`

3. **Default Behavior**:
   - If toggle is OFF or not set: `continue_on_error` is omitted or `false`
   - If toggle is ON: `continue_on_error: true` is added to action object

4. **No Backend Changes Needed**: The property is already:
   - Defined in the schema
   - Validated during config save
   - Persisted to YAML
   - Used during automation execution

### Example YAML Output

When user enables the toggle:
```yaml
automation:
  - id: "123456"
    alias: "Example Automation"
    triggers:
      - trigger: state
        entity_id: binary_sensor.motion
        to: "on"
    actions:
      - action: light.turn_on
        target:
          entity_id: light.bedroom
        continue_on_error: true  # <-- Added by toggle
      - action: notify.mobile_app
        data:
          message: "Light turned on"
```

## Related Research

- `thoughts/shared/research/2025-11-24-spook-port-enable-disable-config-entry.md` - Related config entry research

## Open Questions

1. **Frontend Repository**: The actual UI changes need to be made in `home-assistant/frontend` repo, not this core repo. The file is likely `src/panels/config/automation/action/ha-automation-action-row.ts`

2. **UI Placement**: Where exactly should the toggle appear? Options:
   - In the action card header (like `enabled`)
   - In an expanded options section
   - In a settings menu for the action

3. **User Education**: Should there be a tooltip explaining which errors can be continued from and which cannot?
