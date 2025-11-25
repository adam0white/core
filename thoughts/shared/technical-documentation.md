# Technical Documentation: Home Assistant Core Contributions

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Feature 1: Enable/Disable Config Entry Services](#feature-1-enabledisable-config-entry-services)
3. [Feature 2: Continue on Error Toggle](#feature-2-continue-on-error-toggle)
4. [Setup Instructions](#setup-instructions)
5. [Decision Log](#decision-log)
6. [Test Coverage](#test-coverage)

---

## Architecture Overview

### Home Assistant Core Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                     Home Assistant Core                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   Event Bus  │  │   Services   │  │   State Machine      │   │
│  │  (core.py)   │  │  Registry    │  │   (config_entries)   │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│         │                 │                    │                │
│         ▼                 ▼                    ▼                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Components Layer                     │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  |    │
│  │  │homeassistant│  │ automation  │  │   light, etc.   │  │    │
│  │  │  (core svcs)│  │  (engine)   │  │  (platforms)    │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│         │                 │                    │                │
│         ▼                 ▼                    ▼                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Helpers Layer                       │    │
│  │  config_validation.py │ service.py │ script.py          │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Config Entry Lifecycle

```
┌─────────────┐    async_setup()    ┌─────────────┐
│  NOT_LOADED │ ──────────────────► │   LOADED    │
└─────────────┘                     └─────────────┘
       ▲                                   │
       │    async_unload()                 │
       └───────────────────────────────────┘

       │  async_set_disabled_by(USER)      │
       ▼                                   ▼
┌─────────────┐                     ┌─────────────┐
│  DISABLED   │ ◄─────────────────► │   RELOAD    │
└─────────────┘  async_set_disabled └─────────────┘
                    _by(None)
```

### Service Registration Flow

```
┌────────────────────┐
│  async_setup()     │
│  in __init__.py    │
└─────────┬──────────┘
          │
          ▼
┌────────────────────────────────────┐
│  async_register_admin_service()    │
│  - Wraps handler with admin check  │
│  - Validates schema                │
│  - Registers with ServiceRegistry  │
└─────────┬──────────────────────────┘
          │
          ▼
┌────────────────────┐     ┌─────────────────┐
│  services.yaml     │────►│  UI Action      │
│  (field definitions)     │  Picker         │
└────────────────────┘     └─────────────────┘
          │
          ▼
┌────────────────────┐
│  strings.json      │────► Translations
│  (descriptions)    │
└────────────────────┘
```

---

## Feature 1: Enable/Disable Config Entry Services

### Problem Statement
Users cannot programmatically enable/disable integrations via automations. The only option is manual UI interaction or installing the Spook custom component.

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Service Call Flow                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Automation/Script                                          │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────────────────────┐                    │
│  │ homeassistant.disable_config_entry  │                    │
│  │ data:                               │                    │
│  │   entry_id: "abc123"                │                    │
│  └─────────────────┬───────────────────┘                    │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────┐                    │
│  │ async_handle_disable_config_entry() │                    │
│  │ homeassistant/__init__.py:356       │                    │
│  └─────────────────┬───────────────────┘                    │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────┐                    │
│  │ _async_set_config_entry_disabled_by │                    │
│  │ homeassistant/__init__.py:362       │                    │
│  └─────────────────┬───────────────────┘                    │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────┐                    │
│  │ ConfigEntries.async_set_disabled_by │                    │
│  │ config_entries.py:2300              │                    │
│  │                                     │                    │
│  │ - Sets disabled_by attribute        │                    │
│  │ - Updates device/entity registries  │                    │
│  │ - Triggers reload                   │                    │
│  └─────────────────────────────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Files Modified

| File | Purpose | Lines Changed |
|------|---------|---------------|
| `homeassistant/components/homeassistant/__init__.py` | Service handlers & registration | +30 |
| `homeassistant/components/homeassistant/services.yaml` | UI field definitions | +16 |
| `homeassistant/components/homeassistant/strings.json` | Translations | +18 |
| `tests/components/homeassistant/test_init.py` | Unit tests | +40 |

### API Reference

#### `homeassistant.enable_config_entry`
Enables a previously disabled config entry.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entry_id` | string | Yes | Config entry ID to enable |

#### `homeassistant.disable_config_entry`
Disables a config entry, preventing it from loading.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entry_id` | string | Yes | Config entry ID to disable |

### Example Usage

```yaml
# Automation: Privacy mode when home
automation:
  - alias: "Enable privacy mode"
    trigger:
      - platform: state
        entity_id: person.john
        to: "home"
    action:
      - service: homeassistant.disable_config_entry
        data:
          entry_id: "cloud_integration_entry_id"
      - service: homeassistant.disable_config_entry
        data:
          entry_id: "camera_integration_entry_id"
```

---

## Feature 2: Continue on Error Toggle

### Problem Statement
The `continue_on_error` action property is only accessible via YAML. Visual Editor users cannot toggle it without switching modes.

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Frontend Component Hierarchy                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────┐                    │
│  │     ha-automation-editor            │                    │
│  └─────────────────┬───────────────────┘                    │
│                    │                                        │
│          ┌─────────┴─────────┐                              │
│          ▼                   ▼                              │
│  ┌───────────────┐   ┌───────────────────┐                  │
│  │ Action Row    │   │ Sidebar Panel     │                  │
│  │ (inline edit) │   │ (expanded edit)   │                  │
│  └───────┬───────┘   └─────────┬─────────┘                  │
│          │                     │                            │
│          ▼                     ▼                            │
│  ┌─────────────────────────────────────┐                    │
│  │         Overflow Menu               │                    │
│  │  ┌─────────────────────────────┐    │                    │
│  │  │ ☐ Enabled                   │    │                    │
│  │  │ ☐ Continue on error  ◄──NEW │    │                    │
│  │  │ ─────────────────────       │    │                    │
│  │  │ Rename / Duplicate / Delete │    │                    │
│  │  └─────────────────────────────┘    │                    │
│  └─────────────────────────────────────┘                    │
│                    │                                        │
│                    ▼                                        │
│  ┌─────────────────────────────────────┐                    │
│  │     fireEvent("value-changed")      │                    │
│  │     { continue_on_error: true }     │                    │
│  └─────────────────────────────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User clicks menu item
        │
        ▼
┌─────────────────────────┐
│ _onContinueOnError()    │
│ Toggle boolean value    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ fireEvent(value-changed)│
│ with updated action     │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Parent component        │
│ updates action array    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Badge icon appears      │
│ (visual indicator)      │
└─────────────────────────┘
```

### Files Modified (Frontend Repo)

| File | Purpose | Lines Changed |
|------|---------|---------------|
| `src/panels/config/automation/action/ha-automation-action-row.ts` | Menu item in action row | +20 |
| `src/panels/config/automation/sidebar/ha-automation-sidebar-action.ts` | Menu item in sidebar | +20 |

### Visual Indicator
When `continue_on_error: true`, a badge icon appears on the action card, consistent with other action properties like "disabled".

---

## Setup Instructions

### Prerequisites
- Docker Desktop (for Dev Container)
- VS Code with Remote Containers extension
- Git

### Quick Start (Dev Container - Recommended)

```bash
# 1. Clone the repository
git clone https://github.com/home-assistant/core.git
cd core

# 2. Open in VS Code
code .

# 3. When prompted, click "Reopen in Container"
#    Or: Cmd/Ctrl+Shift+P → "Remote-Containers: Reopen in Container"

# 4. Wait for container to build (~5 minutes first time)

# 5. Run Home Assistant
hass -c config
```

### Testing Feature 1: Enable/Disable Config Entry

```bash
# Run specific tests
pytest tests/components/homeassistant/test_init.py -k "disable_config_entry or enable_config_entry" -v

# Run all homeassistant component tests
pytest tests/components/homeassistant/test_init.py -v

# Run with coverage
pytest tests/components/homeassistant/test_init.py \
  --cov=homeassistant.components.homeassistant \
  --cov-report=term-missing
```

### Manual Testing

1. Start Home Assistant: `hass -c config`
2. Open browser: `http://localhost:8123`
3. Go to Developer Tools → Services
4. Search for `homeassistant.enable_config_entry`
5. Enter a valid config entry ID
6. Click "Call Service"
7. Verify in Settings → Devices & Services

### Finding Config Entry IDs

```yaml
# Option 1: Developer Tools → Services → select any service with config_entry selector
# Option 2: Check .storage/core.config_entries file
# Option 3: Browser DevTools → Network → filter "config_entries"
```

---

## Architectural Constraints & Learnings

During the development of the Integration Management Service, I engaged with core maintainer Franck Nijhof (Frenck). The interaction revealed a distinct architectural boundary in Home Assistant: Service Actions are designed for ephemeral state changes (turning on a light), whereas Configuration Entries represent persistent infrastructure.

Allowing Service Actions to modify Configuration Entries would violate the Principle of Least Surprise; a user restarting the system expects the configuration to be static, not altered by a background automation loop. This explains why such features are delegated to custom components (like Spook) which bypass these safety rails for power users only.

---

## Decision Log

### Feature 1: Enable/Disable Config Entry

| Decision | Options Considered | Choice | Rationale |
|----------|-------------------|--------|-----------|
| **Service naming** | `set_config_entry_disabled` (single) vs `enable/disable` (two) | Two services | Matches Spook API for migration; clearer intent |
| **Parameter name** | `config_entry_id` (Spook) vs `entry_id` (HA Core) | `entry_id` | Consistency with existing `reload_config_entry` service |
| **Schema** | Simple required `entry_id` vs entity/device targeting | Simple schema | Simpler API; targeting adds complexity without clear benefit |
| **Admin requirement** | Regular vs admin-only | Admin-only | Disabling integrations is privileged; matches reload pattern |
| **Error handling** | Silent fail vs raise exception | Raise exception | Let core API handle errors; consistent with reload service |

The PR was rejected because Home Assistant enforces a strict separation between Configuration State (managed by the Config Flow/User) and Runtime State (managed by Automations/Services). Allowing services to modify configuration introduces circular dependencies where an automation could permanently disable its own triggers, leading to 'zombie' states that persist across reboots.

### Feature 2: Continue on Error Toggle

| Decision | Options Considered | Choice | Rationale |
|----------|-------------------|--------|-----------|
| **UI placement** | Separate panel vs overflow menu | Overflow menu | Matches existing "Enabled" toggle pattern |
| **Show for conditions** | All actions vs non-conditions only | Non-conditions only | Conditions have different error semantics |
| **Default state** | Show current value vs always show toggle | Dynamic icon | Filled icon when enabled, outline when disabled |
| **Property removal** | Keep `false` vs delete property | Delete when false | Cleaner YAML; matches other boolean properties |

---

## Test Coverage

### Feature 1: Unit Tests

```
tests/components/homeassistant/test_init.py
├── test_disable_config_entry_by_entry_id    ✓
├── test_enable_config_entry_by_entry_id     ✓
└── test_disable_config_entry_no_match       ✓

Coverage: 3 tests covering:
- Disable calls async_set_disabled_by with ConfigEntryDisabler.USER
- Enable calls async_set_disabled_by with None
- Error raised when no matching entry
```

### Test Commands

```bash
# Run Feature 1 tests
pytest tests/components/homeassistant/test_init.py \
  -k "disable_config_entry or enable_config_entry" -v

# Expected output:
# test_disable_config_entry_by_entry_id PASSED
# test_enable_config_entry_by_entry_id PASSED
# test_disable_config_entry_no_match PASSED
```

### Why Limited Tests?

| Not Tested | Reason |
|------------|--------|
| Entity/device targeting | `async_extract_config_entry_ids` tested elsewhere |
| Core disable logic | `async_set_disabled_by` tested in config_entries tests |
| Multiple entries | Same code path as single entry |

### Feature 2: Frontend Tests

Frontend tests follow different patterns (Playwright/browser-based). The toggle functionality mirrors the existing "Enabled" toggle which has existing test coverage.

---

## PR Links

- **Feature 1**: [home-assistant/core#157216](https://github.com/home-assistant/core/pull/157216)
- **Feature 2**: [home-assistant/frontend#28095](https://github.com/home-assistant/frontend/pull/28095)

---

## Building on This Work

### Extending Feature 1

To add more config entry operations:

1. Add service constant in `__init__.py`
2. Create handler function calling appropriate `ConfigEntries` method
3. Register with `async_register_admin_service`
4. Add to `services.yaml` and `strings.json`
5. Add tests

### Extending Feature 2

To add more action properties to Visual Editor:

1. Find property in `SCRIPT_ACTION_BASE_SCHEMA` (config_validation.py)
2. Add menu item in `ha-automation-action-row.ts`
3. Add toggle method following `_onContinueOnError` pattern
4. Mirror in sidebar component
5. Add/reuse translation string

---

## References

- [Home Assistant Architecture](https://developers.home-assistant.io/docs/architecture_index)
- [Config Entries Documentation](https://developers.home-assistant.io/docs/config_entries_index)
- [Service Registration](https://developers.home-assistant.io/docs/dev_101_services)
- [Frontend Development](https://developers.home-assistant.io/docs/frontend)
