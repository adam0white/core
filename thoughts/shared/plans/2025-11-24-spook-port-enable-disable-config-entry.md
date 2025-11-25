# Enable/Disable Config Entry Services Implementation Plan

## Overview

Port the Spook custom component's `enable_config_entry` and `disable_config_entry` services into Home Assistant Core. This enables users to programmatically enable/disable integrations via automations, supporting use cases like "Privacy Mode" (disable cameras when home) or "Energy Mode" (disable polling integrations).

## Current State Analysis

### What Exists Now
- **Core API**: `hass.config_entries.async_set_disabled_by(entry_id, disabled_by)` in `homeassistant/config_entries.py:2300-2332` already handles the enable/disable logic
- **WebSocket API**: UI uses `config_entries/disable` WebSocket command in `homeassistant/components/config/config_entries.py:553-576`
- **Similar Service**: `reload_config_entry` in `homeassistant/components/homeassistant/__init__.py:337-358` provides the pattern to follow

### What's Missing
- No service call API for automations to enable/disable integrations
- Users must use the UI or install Spook custom component

### Key Constraints
- Must be admin-only services (security requirement)
- Should support both direct `entry_id` and entity/device/area targeting (like `reload_config_entry`)
- Must handle errors gracefully with proper translations

## Desired End State

After implementation:
1. Users can call `homeassistant.enable_config_entry` and `homeassistant.disable_config_entry` from automations
2. Services appear in the Action Picker in the Automation Editor
3. Services support targeting by `entry_id`, `entity_id`, `device_id`, `area_id`, `floor_id`, or `label_id`
4. Proper error messages are shown when entries don't exist or operations fail

### Verification
- Services appear in Developer Tools > Actions
- Calling `homeassistant.disable_config_entry` with a valid entry_id disables the integration
- Calling `homeassistant.enable_config_entry` re-enables it
- Services work from automations
- Non-admin users cannot call the services

## What We're NOT Doing

- NOT adding response data (matching existing service patterns)
- NOT adding a single combined service with a boolean parameter (following Spook's API for migration compatibility)
- NOT modifying the core `async_set_disabled_by` API
- NOT adding WebSocket API changes (already exists)

## Implementation Approach

Follow the exact pattern of `reload_config_entry` service:
1. Define a schema that accepts `entry_id` and/or entity service fields
2. Create handler functions that extract config entry IDs and call the core API
3. Register as admin-only services
4. Add service definitions and translations

---

## Phase 1: Add Service Handlers & Registration

### Overview
Add the service handlers and registration to `homeassistant/components/homeassistant/__init__.py`.

### Changes Required:

#### 1.1 Add Import for ConfigEntryDisabler

**File**: `homeassistant/components/homeassistant/__init__.py`
**Changes**: Add import for `ConfigEntryDisabler` and exceptions

```python
# Add to existing imports from homeassistant (around line 12)
from homeassistant import config as conf_util, config_entries, core_config
```

Note: `config_entries` module import provides access to `ConfigEntryDisabler`, `UnknownEntry`, and `OperationNotAllowed`.

#### 1.2 Add Service Name Constants

**File**: `homeassistant/components/homeassistant/__init__.py`
**Changes**: Add constants after line 82 (after `SERVICE_RELOAD_ALL`)

```python
SERVICE_RELOAD_ALL = "reload_all"
SERVICE_ENABLE_CONFIG_ENTRY = "enable_config_entry"
SERVICE_DISABLE_CONFIG_ENTRY = "disable_config_entry"
```

#### 1.3 Add Service Handler Functions

**File**: `homeassistant/components/homeassistant/__init__.py`
**Changes**: Add handler functions after `async_handle_reload_config_entry` (after line 350)

```python
    async def async_handle_enable_config_entry(call: ServiceCall) -> None:
        """Service handler for enabling a config entry."""
        await _async_set_config_entries_disabled_by(hass, call, disabled_by=None)

    async def async_handle_disable_config_entry(call: ServiceCall) -> None:
        """Service handler for disabling a config entry."""
        await _async_set_config_entries_disabled_by(
            hass, call, disabled_by=config_entries.ConfigEntryDisabler.USER
        )

    async def _async_set_config_entries_disabled_by(
        hass: HomeAssistant,
        call: ServiceCall,
        *,
        disabled_by: config_entries.ConfigEntryDisabler | None,
    ) -> None:
        """Enable or disable config entries."""
        target_entries: set[str] = set()
        if ATTR_ENTRY_ID in call.data:
            target_entries.add(call.data[ATTR_ENTRY_ID])
        target_entries.update(await async_extract_config_entry_ids(call))

        if not target_entries:
            raise ValueError("There were no matching config entries to enable/disable")

        await asyncio.gather(
            *(
                hass.config_entries.async_set_disabled_by(entry_id, disabled_by)
                for entry_id in target_entries
            )
        )
```

#### 1.4 Register the Services

**File**: `homeassistant/components/homeassistant/__init__.py`
**Changes**: Add service registration after `reload_config_entry` registration (after line 358)

```python
    async_register_admin_service(
        hass,
        DOMAIN,
        SERVICE_ENABLE_CONFIG_ENTRY,
        async_handle_enable_config_entry,
        schema=SCHEMA_RELOAD_CONFIG_ENTRY,
    )

    async_register_admin_service(
        hass,
        DOMAIN,
        SERVICE_DISABLE_CONFIG_ENTRY,
        async_handle_disable_config_entry,
        schema=SCHEMA_RELOAD_CONFIG_ENTRY,
    )
```

### Success Criteria:

#### Automated Verification:
- [ ] Linting passes: `python -m script.hassfest --integration-path homeassistant/components/homeassistant`
- [ ] Type checking passes: `mypy homeassistant/components/homeassistant`
- [ ] Existing tests still pass: `pytest tests/components/homeassistant -x`

#### Manual Verification:
- [ ] Services appear in Developer Tools > Actions when Home Assistant is started
- [ ] Calling `homeassistant.disable_config_entry` with a valid entry_id works

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Add Service Definitions

### Overview
Update `services.yaml` to define the new services for the UI Action Picker.

### Changes Required:

#### 2.1 Add Service Definitions

**File**: `homeassistant/components/homeassistant/services.yaml`
**Changes**: Add after `reload_config_entry` definition (after line 60)

```yaml
enable_config_entry:
  target:
  fields:
    entry_id:
      advanced: true
      required: false
      example: 8955375327824e14ba89e4b29cc3ec9a
      selector:
        config_entry:

disable_config_entry:
  target:
  fields:
    entry_id:
      advanced: true
      required: false
      example: 8955375327824e14ba89e4b29cc3ec9a
      selector:
        config_entry:
```

### Success Criteria:

#### Automated Verification:
- [ ] Hassfest validation passes: `python -m script.hassfest --integration-path homeassistant/components/homeassistant`

#### Manual Verification:
- [ ] Services show proper field descriptions in Developer Tools > Actions
- [ ] Config entry selector dropdown appears when selecting the service

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Add Translations

### Overview
Update `strings.json` to add service descriptions and field descriptions.

### Changes Required:

#### 3.1 Add Service Translations

**File**: `homeassistant/components/homeassistant/strings.json`
**Changes**: Add to the `"services"` object (after `reload_config_entry` around line 194)

```json
    "enable_config_entry": {
      "description": "Enables a disabled configuration entry, allowing the integration to be loaded.",
      "fields": {
        "entry_id": {
          "description": "The configuration entry ID of the entry to be enabled.",
          "name": "Config entry ID"
        }
      },
      "name": "Enable config entry"
    },
    "disable_config_entry": {
      "description": "Disables a configuration entry, preventing the integration from being loaded.",
      "fields": {
        "entry_id": {
          "description": "The configuration entry ID of the entry to be disabled.",
          "name": "Config entry ID"
        }
      },
      "name": "Disable config entry"
    },
```

### Success Criteria:

#### Automated Verification:
- [ ] Hassfest validation passes: `python -m script.hassfest --integration-path homeassistant/components/homeassistant`
- [ ] Translations generate correctly: `python -m script.translations develop --integration homeassistant`

#### Manual Verification:
- [ ] Service names and descriptions appear correctly in Developer Tools > Actions

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Add Tests

### Overview
Add tests for the new services following existing patterns in `tests/components/homeassistant/test_init.py`.

### Changes Required:

#### 4.1 Add Import for config_entries in Test File

**File**: `tests/components/homeassistant/test_init.py`
**Changes**: Add import at top of file

```python
from homeassistant import config_entries
```

#### 4.2 Add Test for Disable Config Entry

**File**: `tests/components/homeassistant/test_init.py`
**Changes**: Add test function after `test_reload_config_entry_by_entry_id`

```python
async def test_disable_config_entry_by_entry_id(hass: HomeAssistant) -> None:
    """Test being able to disable a config entry by config entry id."""
    await async_setup_component(hass, "homeassistant", {})

    with patch(
        "homeassistant.config_entries.ConfigEntries.async_set_disabled_by",
        return_value=True,
    ) as mock_disable:
        await hass.services.async_call(
            "homeassistant",
            "disable_config_entry",
            {ATTR_ENTRY_ID: "8955375327824e14ba89e4b29cc3ec9a"},
            blocking=True,
        )

    assert len(mock_disable.mock_calls) == 1
    assert mock_disable.mock_calls[0][1][0] == "8955375327824e14ba89e4b29cc3ec9a"
    assert mock_disable.mock_calls[0][1][1] == config_entries.ConfigEntryDisabler.USER
```

#### 4.3 Add Test for Enable Config Entry

**File**: `tests/components/homeassistant/test_init.py`
**Changes**: Add test function

```python
async def test_enable_config_entry_by_entry_id(hass: HomeAssistant) -> None:
    """Test being able to enable a config entry by config entry id."""
    await async_setup_component(hass, "homeassistant", {})

    with patch(
        "homeassistant.config_entries.ConfigEntries.async_set_disabled_by",
        return_value=True,
    ) as mock_enable:
        await hass.services.async_call(
            "homeassistant",
            "enable_config_entry",
            {ATTR_ENTRY_ID: "8955375327824e14ba89e4b29cc3ec9a"},
            blocking=True,
        )

    assert len(mock_enable.mock_calls) == 1
    assert mock_enable.mock_calls[0][1][0] == "8955375327824e14ba89e4b29cc3ec9a"
    assert mock_enable.mock_calls[0][1][1] is None
```

#### 4.4 Add Test for No Matching Entries Error

**File**: `tests/components/homeassistant/test_init.py`
**Changes**: Add test function

```python
async def test_disable_config_entry_no_match(hass: HomeAssistant) -> None:
    """Test error when no matching config entries found."""
    await async_setup_component(hass, "homeassistant", {})

    with pytest.raises(ValueError):
        await hass.services.async_call(
            "homeassistant",
            "disable_config_entry",
            {"entity_id": "unknown.entity_id"},
            blocking=True,
        )
```

### Success Criteria:

#### Automated Verification:
- [ ] All new tests pass: `pytest tests/components/homeassistant/test_init.py -k "disable_config_entry or enable_config_entry" -v`
- [ ] All existing tests still pass: `pytest tests/components/homeassistant/test_init.py -v`

#### Manual Verification:
- [ ] End-to-end test: Create an automation that disables/enables a real integration

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding.

---

## Testing Strategy

### Unit Tests (3 tests):
- Test disable by `entry_id` (verifies `ConfigEntryDisabler.USER` is passed)
- Test enable by `entry_id` (verifies `None` is passed)
- Test error case when no matching entries found

### Why Only 3 Tests:
- Entity/device/area targeting uses `async_extract_config_entry_ids` which is already tested elsewhere
- The core `async_set_disabled_by` functionality is tested in config_entries tests

### Manual Testing Steps:
1. Start Home Assistant with the changes
2. Go to Developer Tools > Actions
3. Find `homeassistant.enable_config_entry` and `homeassistant.disable_config_entry`
4. Test disabling a real integration
5. Verify the integration appears disabled in Settings > Devices & Services
6. Test enabling it again

## Performance Considerations

- `async_set_disabled_by` already handles all the heavy lifting (reload, registry updates)
- Multiple entries are processed concurrently with `asyncio.gather()`
- No additional performance concerns

## Migration Notes

- Users of Spook's services can switch to the native services with a minor change
- **Parameter name difference**: Spook uses `config_entry_id`, we use `entry_id` to match the existing `reload_config_entry` service in HA Core
- **Improved targeting**: Unlike Spook, our implementation also supports entity/device/area targeting (same as `reload_config_entry`)
- Migration example:
  ```yaml
  # Spook (old)
  action: homeassistant.disable_config_entry
  data:
    config_entry_id: "abc123"

  # HA Core (new)
  action: homeassistant.disable_config_entry
  data:
    entry_id: "abc123"
  ```

## References

- Research document: `thoughts/shared/research/2025-11-24-spook-port-enable-disable-config-entry.md`
- Spook implementation: [https://github.com/frenck/spook](https://github.com/frenck/spook)
- Core API: `homeassistant/config_entries.py:2300-2332`
- Similar service pattern: `homeassistant/components/homeassistant/__init__.py:337-358`
