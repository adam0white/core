# Home Assistant Core Contributions - Video Script

**Duration**: 7-10 minutes
**Repository**: [home-assistant/core](https://github.com/home-assistant/core)

---

## 1. Introduction (1 minute)

### Why Home Assistant?

"I chose Home Assistant Core for several reasons:

1. **Massive, mature codebase** - Over 10 years old, 4000+ integrations, millions of users. This isn't a toy project - it's production software running in homes worldwide.

2. **Python expertise** - As someone comfortable with Python, I wanted to contribute to a project where I could go deep, not just scratch the surface.

3. **Real user impact** - Every feature I build gets used by real people automating their homes. The feedback loop is immediate and tangible.

4. **Quick Development** - Home Assistant has embraced Dev Containers (VS Code Remote Containers). A contributor can clone the repo, open it in VS Code, and the IDE will automatically build a Docker container with all Python dependencies, ffmpeg, and system tools pre-installed. This reduces setup time to minutes

---

## 2. Codebase Structure (1.5 minutes)

### High-Level Architecture

"Let me walk you through how Home Assistant is organized:

```
homeassistant/
├── core.py              # Event loop, service registry, state machine
├── config_entries.py    # Integration lifecycle management
├── const.py             # Global constants
├── components/          # 4000+ integrations live here
│   ├── homeassistant/   # Core services (restart, reload, turn_on/off)
│   ├── light/           # Light platform
│   ├── automation/      # Automation engine
│   └── ...
├── helpers/
│   ├── config_validation.py  # Schema validation (voluptuous)
│   ├── service.py            # Service registration helpers
│   └── script.py             # Action execution engine
└── ...

tests/
└── components/          # Mirrors the components structure
```

### Key Patterns

- **Config Entries**: Every integration instance is a 'config entry' with a unique ID, state machine (LOADED, NOT_LOADED, etc.), and lifecycle methods
- **Services**: Actions users can call - registered with schemas, handlers, and optional admin-only restrictions
- **Coordinators**: Centralized data fetching to avoid hammering APIs

The pattern I followed: find where similar features exist, understand the pattern, apply it to my feature."

---

## 3. Feature 1: Enable/Disable Config Entry Services (3 minutes)

### The Problem

"Currently, if you want to disable an integration - say, turn off your cameras when you're home for privacy, or disable Christmas lights in July - you have to:

1. Open Settings
2. Navigate to Devices & Services
3. Find the integration
4. Click the three-dot menu
5. Click Disable

**You cannot automate this.** No service call exists. Users have been asking for this, and Frenck built it into Spook as `homeassistant.enable_config_entry` and `homeassistant.disable_config_entry`."

### My Design Approach

"The beauty of this feature: **the core API already exists**.

```python
# homeassistant/config_entries.py:2300
async def async_set_disabled_by(self, entry_id, disabled_by):
    # Handles everything: state changes, registry updates, reload
```

The UI uses this via WebSocket. I just needed to expose it as a service.

**Key design decisions:**

1. **Two services vs one** - I chose two (`enable_config_entry`, `disable_config_entry`) to match Spook's API for easy migration, rather than a single `set_config_entry_disabled` with a boolean.

2. **Parameter naming** - Spook uses `config_entry_id`, but I used `entry_id` to match the existing `reload_config_entry` service. Consistency within HA Core matters more than matching Spook exactly.

3. **Admin-only** - Used `async_register_admin_service()` because disabling integrations is a privileged operation."

### Demo

*[Show Developer Tools > Services]*

"Here are the new services. I can disable my test integration..."

*[Call disable_config_entry with entry_id]*

"...and it's now disabled in the UI. I can re-enable it..."

*[Call enable_config_entry]*

"...and it's back. This works in automations too - imagine an automation that disables your camera integration when you arrive home."

### Code Walkthrough

"Let me show you the implementation:

```python
# homeassistant/components/homeassistant/__init__.py

# 1. Define constants
SERVICE_ENABLE_CONFIG_ENTRY = "enable_config_entry"
SERVICE_DISABLE_CONFIG_ENTRY = "disable_config_entry"

# 2. Simple schema - just requires entry_id
SCHEMA_CONFIG_ENTRY_ID = vol.Schema({vol.Required(ATTR_ENTRY_ID): str})

# 3. Handlers delegate to the shared helper
async def async_handle_enable_config_entry(call: ServiceCall) -> None:
    await _async_set_config_entry_disabled_by(hass, call, disabled_by=None)

async def async_handle_disable_config_entry(call: ServiceCall) -> None:
    await _async_set_config_entry_disabled_by(
        hass, call, disabled_by=config_entries.ConfigEntryDisabler.USER
    )

# 4. Shared helper calls the core API
async def _async_set_config_entry_disabled_by(hass, call, *, disabled_by):
    entry_id = call.data[ATTR_ENTRY_ID]
    await hass.config_entries.async_set_disabled_by(entry_id, disabled_by)

# 5. Register as admin services
async_register_admin_service(
    hass, DOMAIN, SERVICE_ENABLE_CONFIG_ENTRY,
    async_handle_enable_config_entry,
    schema=SCHEMA_CONFIG_ENTRY_ID,
)
```

The services.yaml and strings.json provide UI metadata and translations."

### Challenges

"The main challenge was **deciding on the API surface**. The existing `reload_config_entry` supports entity/device/area targeting - should enable/disable too?

I initially planned to match that pattern, but realized it adds complexity without clear benefit. If you want to disable an integration, you know its entry_id. Simpler is better."

---

## 4. Feature 2: Continue on Error Toggle (3 minutes)

### The Problem

"When an automation runs multiple actions and one fails, the whole automation stops. Sometimes that's what you want. But often, you want it to continue - maybe the notification service is down, but you still want the lights to turn on.

Home Assistant has a `continue_on_error` property that does exactly this. **But it's only accessible in YAML mode.** Visual Editor users can't use it without switching to YAML."

### Research Phase

"I started by researching where this property lives:

```python
# homeassistant/helpers/config_validation.py:1463
SCRIPT_ACTION_BASE_SCHEMA = {
    vol.Optional(CONF_ALIAS): string,
    vol.Optional(CONF_CONTINUE_ON_ERROR): boolean,  # <-- Already here!
    vol.Optional(CONF_ENABLED): vol.Any(boolean, template),
}
```

The backend fully supports it. The frontend data model has it too:

```typescript
// frontend/src/data/script.ts:108
continue_on_error?: boolean
```

There's even a visual indicator that shows when it's enabled! The only missing piece: a menu item to toggle it."

### My Design Approach

"This is a **frontend-only change**. No backend work needed.

I followed the exact pattern of the 'Enable/Disable' toggle that already exists:

1. Add menu item to action overflow menu
2. Add toggle method that fires `value-changed` event
3. Mirror in sidebar component for consistency

**Key decision**: Only show for non-condition actions. Conditions have different error semantics - they don't 'fail' the same way service calls do."

### Demo

*[Open automation editor, add action]*

"I'll add a light turn_on action. Now in the three-dot menu..."

*[Click menu]*

"...there's 'Continue on error'. I click it..."

*[Toggle it on]*

"...and now this icon appears showing it's enabled. If I switch to YAML mode..."

*[Switch to YAML]*

"...you can see `continue_on_error: true` in the action. No more manual YAML editing for this common need."

### Code Walkthrough

"The implementation mirrors the existing 'Enable/Disable' pattern:

```typescript
// ha-automation-action-row.ts

// Menu item with dynamic icon
${type !== "condition" ? html`
  <ha-md-menu-item .clickAction=${this._onContinueOnError}>
    <ha-svg-icon
      slot="start"
      .path=${action.continue_on_error === true
        ? mdiAlertCircleCheck      // Filled when enabled
        : mdiAlertCircleCheckOutline}  // Outline when disabled
    ></ha-svg-icon>
    ${this.hass.localize(
      "ui.panel.config.automation.editor.actions.continue_on_error"
    )}
  </ha-md-menu-item>
` : nothing}

// Toggle method
private _onContinueOnError = () => {
  const continueOnError = !(this.action.continue_on_error ?? false);
  const value = continueOnError
    ? { ...this.action, continue_on_error: true }
    : { ...this.action };
  if (!continueOnError) {
    delete value.continue_on_error;  // Remove property when false
  }
  fireEvent(this, "value-changed", { value });
};
```

The translation string already existed - I just reused it."

### Challenges

"The challenge was **understanding the component architecture**. Home Assistant's frontend has:

- Action row component (inline editing)
- Sidebar component (panel editing)
- Both need the menu item

I had to trace through the event system to understand how `value-changed` propagates up and how the sidebar gets refreshed."

---

## 5. PRs and Maintainer Discussions (30 seconds)

"Links to my PRs:

- **Enable/Disable Config Entry**: [PR #157216](https://github.com/home-assistant/core/pull/157216)
- **Continue on Error Toggle**: [PR #28095](https://github.com/home-assistant/frontend/pull/28095) (frontend repo)

*[There are maintainer comments, discuss them]*

---

## 6. What I Learned & Would Do Differently (1 minute)

### What I Learned

1. **Research first, code second** - I spent some time reading existing code before writing any. The `reload_config_entry` service was my template for Feature 1. The `enabled` toggle was my template for Feature 2.

2. **The API often already exists** - Both features were about *exposing* existing functionality, not building new logic. The hardest part is finding where things live.

3. **Consistency trumps cleverness** - I initially wanted to add entity/device targeting to the enable/disable services. But simpler matched user needs better.

4. **Translation strings are critical** - Home Assistant is used globally. Every user-facing string needs proper localization.

### What I'd Do Differently

1. **Start smaller** - My initial plan for Feature 1 had 6 tests. A reviewer would rightfully ask "why so many?" I trimmed to 3.

2. **Check the frontend repo earlier** - For Feature 2, I spent time in the core repo before realizing it was frontend-only. Reading the research docs saved time, but I should have validated sooner.

3. **Engage maintainers earlier** - Opening a draft PR or discussion before coding would have caught API design questions sooner.

---

## Closing

"These contributions taught me that contributing to a large open-source project is less about writing clever code and more about understanding existing patterns, making pragmatic design decisions, and communicating clearly with maintainers.

The code I wrote is simple. The thinking behind it is what matters."

---

## Appendix: File References

### Feature 1: Enable/Disable Config Entry
- `homeassistant/components/homeassistant/__init__.py` - Service handlers
- `homeassistant/components/homeassistant/services.yaml` - Service definitions
- `homeassistant/components/homeassistant/strings.json` - Translations
- `homeassistant/config_entries.py:2300-2332` - Core API
- `tests/components/homeassistant/test_init.py` - Tests

### Feature 2: Continue on Error Toggle
- `frontend/src/panels/config/automation/action/ha-automation-action-row.ts` - Menu item
- `frontend/src/panels/config/automation/sidebar/ha-automation-sidebar-action.ts` - Sidebar menu
- `frontend/src/data/script.ts` - Data model (already had property)
- `homeassistant/helpers/script.py:491-579` - Backend execution (no changes needed)
