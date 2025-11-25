---
date: 2025-11-24T23:53:10Z
researcher: Claude
git_commit: 15647f2720de6029d2de00c145003c0b77bc2982
branch: dev
repository: core
topic: "Port Spook enable/disable config entry services to Home Assistant Core"
tags: [research, codebase, config-entries, services, spook, homeassistant-component]
status: complete
last_updated: 2025-11-24
last_updated_by: Claude
---

# Research: Port Spook Enable/Disable Config Entry Services to Home Assistant Core

**Date**: 2025-11-24T23:53:10Z
**Researcher**: Claude
**Git Commit**: 15647f2720de6029d2de00c145003c0b77bc2982
**Branch**: dev
**Repository**: core

## Research Question

How to port the Spook custom component's `enable_config_entry` and `disable_config_entry` services into Home Assistant Core, enabling users to programmatically enable/disable integrations via automations.

## Summary

The implementation path is straightforward:

1. **The core API already exists**: `hass.config_entries.async_set_disabled_by(entry_id, disabled_by)` in `homeassistant/config_entries.py:2300-2332`
2. **The service registration pattern exists**: Similar to `reload_config_entry` in `homeassistant/components/homeassistant/__init__.py:337-350`
3. **Spook's approach**: Two separate services (`enable_config_entry` and `disable_config_entry`) that accept `config_entry_id` parameter
4. **Alternative approach**: Single service with an `enabled` boolean parameter (similar to entity enable/disable pattern)

## Detailed Findings

### 1. Core API: async_set_disabled_by

**Location**: `homeassistant/config_entries.py:2300-2332`

This is the key method that enables/disables config entries:

```python
async def async_set_disabled_by(
    self, entry_id: str, disabled_by: ConfigEntryDisabler | None
) -> bool:
    """Disable an entry.

    If disabled_by is changed, the config entry will be reloaded.
    """
```

**Implementation Flow**:
1. Gets the entry via `async_get_known_entry(entry_id)` (raises `UnknownEntry` if not found)
2. Validates `disabled_by` parameter
3. Short-circuits if `disabled_by` hasn't changed (returns True)
4. Sets `entry.disabled_by` attribute
5. Schedules storage save
6. Updates device and entity registries
7. Triggers reload via `async_reload(entry_id)`
8. Returns reload result (True = success, False = requires restart)

**Usage**:
- **To enable**: `await hass.config_entries.async_set_disabled_by(entry_id, None)`
- **To disable**: `await hass.config_entries.async_set_disabled_by(entry_id, ConfigEntryDisabler.USER)`

**Related Enum**: `homeassistant/config_entries.py:226-230`
```python
class ConfigEntryDisabler(StrEnum):
    """What disabled a config entry."""
    USER = "user"
```

### 2. Existing WebSocket API (UI Implementation)

**Location**: `homeassistant/components/config/config_entries.py:553-576`

The UI already uses this via WebSocket:

```python
@websocket_api.require_admin
@websocket_api.websocket_command({
    "type": "config_entries/disable",
    "entry_id": str,
    vol.Optional("disabled_by"): vol.Any(
        None, vol.In([e.value for e in config_entries.ConfigEntryDisabler])
    ),
})
async def config_entry_disable(hass, connection, msg):
    """Disable config entry."""
    disabled_by = msg.get("disabled_by")
    if disabled_by is not None:
        disabled_by = config_entries.ConfigEntryDisabler(disabled_by)

    success = await hass.config_entries.async_set_disabled_by(
        msg["entry_id"], disabled_by
    )
    connection.send_result(msg["id"], {"require_restart": not success})
```

### 3. Spook Implementation

**Source**: [Spook GitHub](https://github.com/frenck/spook) - [Documentation](https://spook.boo/integrations/)

Spook provides two services:

#### homeassistant.enable_config_entry
- **Purpose**: Enable a single instance of an integration
- **Parameters**: `config_entry_id` (required, string or list of strings)
- **Example**:
  ```yaml
  action: homeassistant.enable_config_entry
  data:
    config_entry_id: "dc23e666e6100f184e642a0ac345d3eb"
  ```

#### homeassistant.disable_config_entry
- **Purpose**: Disable a single instance of an integration
- **Parameters**: `config_entry_id` (required, string or list of strings)
- **Example**:
  ```yaml
  action: homeassistant.disable_config_entry
  data:
    config_entry_id:
      - "dc23e666e6100f184e642a0ac345d3eb"
      - "df98a97c9341a0f184e642a0ac345d3b"
  ```

### 4. Existing Service Registration Patterns

#### Pattern A: reload_config_entry Service

**Location**: `homeassistant/components/homeassistant/__init__.py:84-92, 337-350`

Schema definition:
```python
SCHEMA_RELOAD_CONFIG_ENTRY = vol.All(
    vol.Schema({
        vol.Optional(ATTR_ENTRY_ID): str,
        **cv.ENTITY_SERVICE_FIELDS,  # device_id, entity_id, area_id, etc.
    }),
    cv.has_at_least_one_key(ATTR_ENTRY_ID, *cv.ENTITY_SERVICE_FIELDS),
)
```

Handler:
```python
async def async_handle_reload_config_entry(call: ServiceCall) -> None:
    reload_entries: set[str] = set()
    if ATTR_ENTRY_ID in call.data:
        reload_entries.add(call.data[ATTR_ENTRY_ID])
    reload_entries.update(await async_extract_config_entry_ids(call))

    if not reload_entries:
        raise ValueError("There were no matching config entries to reload")

    await asyncio.gather(*(
        hass.config_entries.async_reload(entry_id)
        for entry_id in reload_entries
    ))
```

#### Pattern B: Admin Service Registration

**Location**: `homeassistant/helpers/service.py:945-970`

```python
async_register_admin_service(
    hass,
    DOMAIN,
    SERVICE_NAME,
    handler_func,
    schema=SERVICE_SCHEMA,
)
```

This automatically adds admin permission checks.

### 5. Helper Functions

#### async_extract_config_entry_ids

**Location**: `homeassistant/helpers/service.py:465-490`

```python
async def async_extract_config_entry_ids(
    service_call: ServiceCall, expand_group: bool = True
) -> set[str]:
    """Extract referenced config entry ids from a service call."""
```

Extracts config entry IDs from entity_id, device_id, area_id, floor_id, or label_id selectors.

### 6. Constants and Attributes

**Location**: `homeassistant/const.py:338`
```python
ATTR_CONFIG_ENTRY_ID: Final = "config_entry_id"
```

**Location**: `homeassistant/components/homeassistant/__init__.py:72`
```python
ATTR_ENTRY_ID = "entry_id"
```

### 7. Error Handling Patterns

**Exceptions to handle**:
- `config_entries.UnknownEntry`: Entry ID not found
- `config_entries.OperationNotAllowed`: Entry in non-recoverable state

**Pattern**:
```python
try:
    success = await hass.config_entries.async_set_disabled_by(entry_id, disabled_by)
except config_entries.UnknownEntry:
    raise ServiceValidationError(
        translation_domain=DOMAIN,
        translation_key="config_entry_not_found",
    )
except config_entries.OperationNotAllowed:
    raise ServiceValidationError(
        translation_domain=DOMAIN,
        translation_key="config_entry_operation_not_allowed",
    )
```

## Code References

### Primary Files to Modify

- `homeassistant/components/homeassistant/__init__.py` - Service registration
- `homeassistant/components/homeassistant/services.yaml` - Service definitions for UI
- `homeassistant/components/homeassistant/strings.json` - Translation strings

### Reference Files

- `homeassistant/config_entries.py:2300-2332` - `async_set_disabled_by` method
- `homeassistant/config_entries.py:226-230` - `ConfigEntryDisabler` enum
- `homeassistant/components/config/config_entries.py:553-576` - WebSocket API reference
- `homeassistant/helpers/service.py:465-490` - `async_extract_config_entry_ids` helper
- `homeassistant/helpers/service.py:945-970` - `async_register_admin_service` helper

## Architecture Documentation

### Implementation Approaches

#### Approach A: Two Separate Services (Spook Style)
- `homeassistant.enable_config_entry`
- `homeassistant.disable_config_entry`

**Pros**: Clear intent, simple schema, matches Spook for migration
**Cons**: Two services to maintain

#### Approach B: Single Service with Boolean
- `homeassistant.set_config_entry_disabled`

**Pros**: Single service, more flexible
**Cons**: Less intuitive naming

#### Approach C: Extend reload_config_entry Schema
Add `disabled` boolean to existing reload service

**Pros**: Minimal API surface
**Cons**: Conflates reload and disable operations

### Recommended Implementation

Based on existing patterns, **Approach A** (two services) is recommended:

1. Matches Spook's existing API for easy migration
2. Follows the explicit naming pattern used elsewhere in HA
3. Admin-only services (use `async_register_admin_service`)
4. Support both direct `entry_id` and entity/device/area targeting (like reload_config_entry)

### Proposed Service Schema

```python
SCHEMA_ENABLE_DISABLE_CONFIG_ENTRY = vol.All(
    vol.Schema({
        vol.Optional(ATTR_ENTRY_ID): str,
        **cv.ENTITY_SERVICE_FIELDS,
    }),
    cv.has_at_least_one_key(ATTR_ENTRY_ID, *cv.ENTITY_SERVICE_FIELDS),
)
```

### Proposed Handler

```python
async def async_handle_enable_config_entry(call: ServiceCall) -> None:
    """Handle calls to enable_config_entry service."""
    await _async_set_config_entries_disabled(call, disabled=False)

async def async_handle_disable_config_entry(call: ServiceCall) -> None:
    """Handle calls to disable_config_entry service."""
    await _async_set_config_entries_disabled(call, disabled=True)

async def _async_set_config_entries_disabled(
    call: ServiceCall, *, disabled: bool
) -> None:
    """Enable or disable config entries."""
    entries: set[str] = set()
    if ATTR_ENTRY_ID in call.data:
        entries.add(call.data[ATTR_ENTRY_ID])
    entries.update(await async_extract_config_entry_ids(call))

    if not entries:
        raise ServiceValidationError(
            translation_domain=DOMAIN,
            translation_key="no_config_entries_found",
        )

    disabled_by = ConfigEntryDisabler.USER if disabled else None

    results = await asyncio.gather(
        *(
            hass.config_entries.async_set_disabled_by(entry_id, disabled_by)
            for entry_id in entries
        ),
        return_exceptions=True,
    )

    # Handle errors appropriately
```

## Related Research

None yet in this repository.

## Open Questions

1. **Service naming**: Should it be `enable_config_entry`/`disable_config_entry` (Spook style) or follow a different convention?

2. **Response data**: Should the service return information about which entries required restart?

3. **Batch behavior**: What happens if some entries succeed and others fail in a batch operation?

4. **Integration with entity/device targeting**: The reload_config_entry service already supports this pattern - should enable/disable also support it?

5. **Translations**: Need to add entries to `strings.json` for error messages.

## External Resources

- [Spook GitHub Repository](https://github.com/frenck/spook)
- [Spook Documentation](https://spook.boo/)
- [Spook Integration Management Docs](https://spook.boo/integrations/)
- [Home Assistant Community - Spook Discussion](https://community.home-assistant.io/t/spook-your-homie/539588)
