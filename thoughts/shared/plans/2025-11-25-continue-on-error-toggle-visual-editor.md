# Continue on Error Toggle in Visual Editor - Implementation Plan

## Overview

Add a "Continue on error" toggle menu item to the automation action card in the Visual Editor. This allows users to set `continue_on_error: true` on individual actions without switching to YAML mode. The backend already fully supports this property - this is a frontend-only change.

## Current State Analysis

### What Already Exists

1. **Backend Support** (in `core` repo - no changes needed):
   - Schema: `homeassistant/helpers/config_validation.py:1463-1467`
   - Execution: `homeassistant/helpers/script.py:491-579`
   - Property validated and persisted automatically

2. **Frontend Data Model** (already complete):
   - `src/data/script.ts:39` - `continue_on_error: optional(boolean())` in struct
   - `src/data/script.ts:108` - `continue_on_error?: boolean` in interface

3. **Translation String** (already exists):
   - `src/translations/en.json:4470` - `"continue_on_error": "Continue on error"`

4. **Visual Indicator** (already exists):
   - `src/panels/config/automation/action/ha-automation-action-row.ts:277-289`
   - Shows `mdiAlertCircleCheck` icon when `continue_on_error === true`

### What's Missing

- Menu item to toggle `continue_on_error` in the action overflow menu
- Same menu item in the sidebar component
- Callback in `ActionSidebarConfig` interface

## Desired End State

After implementation:
1. Users can click the three-dot menu on any action card
2. A "Continue on error" menu item appears (with checkmark icon when enabled)
3. Clicking toggles `continue_on_error` between `true` and `undefined`/`false`
4. The existing icon indicator in the action row header shows when enabled
5. Works identically in both inline and sidebar editing modes

### Verification
- Create an automation with multiple actions
- Toggle "Continue on error" on one action via menu
- Save and verify YAML shows `continue_on_error: true`
- Reload and verify the toggle state persists

## What We're NOT Doing

- No backend changes (already fully implemented)
- No new translation strings for toggle labels (reusing existing "continue_on_error" string)
- No visual indicator changes (icon already shows when enabled)
- No changes to condition actions (continue_on_error only applies to non-condition actions)

## Implementation Approach

Follow the exact pattern used for the "Enable/Disable" menu item, which:
1. Uses `ha-md-menu-item` with dynamic icon
2. Calls a method that toggles the boolean property
3. Fires `value-changed` event with updated action object
4. Refreshes sidebar if open

---

## Phase 1: Add Menu Item to Action Row Component

### Overview
Add the "Continue on error" toggle menu item to the action row's overflow menu, following the existing "Enable/Disable" pattern.

### Changes Required:

#### 1.1 Add Menu Item to Overflow Menu

**File**: `/Users/abdul/Downloads/Projects/frontend/src/panels/config/automation/action/ha-automation-action-row.ts`

**Location**: After the "Enable/Disable" menu item (around line 446), before the delete menu item.

**Changes**: Add a new menu item that toggles `continue_on_error`. Only show for non-condition actions (since conditions don't support this property).

```typescript
// Add after the enable/disable menu item (line 446) and before the delete item (line 447)
${type !== "condition"
  ? html`
      <ha-md-menu-item
        .clickAction=${this._onContinueOnError}
        .disabled=${this.disabled}
      >
        <ha-svg-icon
          slot="start"
          .path=${(this.action as NonConditionAction).continue_on_error === true
            ? mdiAlertCircleCheck
            : mdiAlertCircleCheckOutline}
        ></ha-svg-icon>

        ${this._renderOverflowLabel(
          this.hass.localize(
            "ui.panel.config.automation.editor.actions.continue_on_error"
          )
        )}
      </ha-md-menu-item>
    `
  : nothing}
```

#### 1.2 Add Icon Import

**File**: `/Users/abdul/Downloads/Projects/frontend/src/panels/config/automation/action/ha-automation-action-row.ts`

**Location**: Import section at top of file (around line 3)

**Changes**: Add `mdiAlertCircleCheckOutline` to imports (note: `mdiAlertCircleCheck` is already imported)

```typescript
import {
  mdiAlertCircleCheck,
  mdiAlertCircleCheckOutline,  // Add this
  mdiAppleKeyboardCommand,
  // ... rest of imports
} from "@mdi/js";
```

#### 1.3 Add Toggle Method

**File**: `/Users/abdul/Downloads/Projects/frontend/src/panels/config/automation/action/ha-automation-action-row.ts`

**Location**: After the `_onDisable` method (around line 610)

**Changes**: Add new method following the same pattern as `_onDisable`

```typescript
private _onContinueOnError = () => {
  const continueOnError = !(
    (this.action as NonConditionAction).continue_on_error ?? false
  );
  const value = continueOnError
    ? { ...this.action, continue_on_error: true }
    : { ...this.action };
  if (!continueOnError) {
    delete (value as NonConditionAction).continue_on_error;
  }
  fireEvent(this, "value-changed", { value });

  if (this._selected && this.optionsInSidebar) {
    this.openSidebar(value);
  }

  if (this._yamlMode && !this.optionsInSidebar) {
    this._actionEditor?.yamlEditor?.setValue(value);
  }
};
```

#### 1.4 Update Sidebar Config Interface

**File**: `/Users/abdul/Downloads/Projects/frontend/src/data/automation.ts`

**Location**: Find `ActionSidebarConfig` interface

**Changes**: Add `continueOnError` callback

```typescript
export interface ActionSidebarConfig {
  // ... existing properties
  continueOnError: () => void;  // Add this
}
```

#### 1.5 Pass Callback to Sidebar

**File**: `/Users/abdul/Downloads/Projects/frontend/src/panels/config/automation/action/ha-automation-action-row.ts`

**Location**: In the `openSidebar` method (around line 798-832)

**Changes**: Add `continueOnError` to the sidebar config object

```typescript
fireEvent(this, "open-sidebar", {
  // ... existing properties
  continueOnError: this._onContinueOnError,  // Add this line
  config: {
    action: sidebarAction,
  },
  // ... rest of properties
} satisfies ActionSidebarConfig);
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd /Users/abdul/Downloads/Projects/frontend && npm run lint`
- [ ] No ESLint errors: `cd /Users/abdul/Downloads/Projects/frontend && npm run lint`

#### Manual Verification:
- [ ] Menu item appears in action overflow menu (three-dot button)
- [ ] Menu item shows outline icon when `continue_on_error` is false/undefined
- [ ] Menu item shows filled icon when `continue_on_error` is true
- [ ] Clicking toggles the property
- [ ] Header icon indicator updates when toggled
- [ ] Menu item does NOT appear for condition actions

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to Phase 2.

---

## Phase 2: Add Menu Item to Sidebar Component

### Overview
Mirror the menu item in the sidebar action component for consistency when editing actions in the sidebar.

### Changes Required:

#### 2.1 Add Icon Import

**File**: `/Users/abdul/Downloads/Projects/frontend/src/panels/config/automation/sidebar/ha-automation-sidebar-action.ts`

**Location**: Import section at top of file

**Changes**: Add the required icons

```typescript
import {
  mdiAlertCircleCheck,
  mdiAlertCircleCheckOutline,
  mdiAppleKeyboardCommand,
  // ... rest of imports
} from "@mdi/js";
```

#### 2.2 Add Menu Item to Sidebar

**File**: `/Users/abdul/Downloads/Projects/frontend/src/panels/config/automation/sidebar/ha-automation-sidebar-action.ts`

**Location**: After the "Enable/Disable" menu item (around line 251), before the delete menu item

**Changes**: Add menu item, checking action type first

```typescript
// Add after the enable/disable menu item and before delete
${getAutomationActionType(actionConfig) !== "condition"
  ? html`
      <ha-md-menu-item
        slot="menu-items"
        .clickAction=${this.config.continueOnError}
        .disabled=${this.disabled}
      >
        <ha-svg-icon
          slot="start"
          .path=${(actionConfig as NonConditionAction).continue_on_error === true
            ? mdiAlertCircleCheck
            : mdiAlertCircleCheckOutline}
        ></ha-svg-icon>
        <div class="overflow-label">
          ${this.hass.localize(
            "ui.panel.config.automation.editor.actions.continue_on_error"
          )}
          <span class="shortcut-placeholder ${isMac ? "mac" : ""}"></span>
        </div>
      </ha-md-menu-item>
    `
  : nothing}
```

#### 2.3 Add Type Import

**File**: `/Users/abdul/Downloads/Projects/frontend/src/panels/config/automation/sidebar/ha-automation-sidebar-action.ts`

**Location**: Import section

**Changes**: Add `NonConditionAction` to imports from script.ts

```typescript
import type { RepeatAction, ServiceAction, NonConditionAction } from "../../../../data/script";
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd /Users/abdul/Downloads/Projects/frontend && npm run lint`
- [ ] No ESLint errors: `cd /Users/abdul/Downloads/Projects/frontend && npm run lint`

#### Manual Verification:
- [ ] Menu item appears in sidebar action menu
- [ ] Icons update correctly based on state
- [ ] Clicking toggles the property and updates the action
- [ ] Menu item does NOT appear for condition actions in sidebar

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Unit Tests

### Overview
Add unit tests for the continue_on_error toggle logic, following the existing Vitest test patterns in the frontend codebase.

### Changes Required:

#### 3.1 Create Test File

**File**: `/Users/abdul/Downloads/Projects/frontend/test/panels/config/automation/action/continue-on-error-toggle.test.ts`

**Changes**: Create new test file for the toggle logic

```typescript
import { describe, it, expect } from "vitest";
import type { NonConditionAction } from "../../../../../src/data/script";

/**
 * Helper function that mirrors the toggle logic from ha-automation-action-row.ts
 * This tests the core logic without needing to instantiate the full component.
 */
function toggleContinueOnError(action: NonConditionAction): NonConditionAction {
  const continueOnError = !(action.continue_on_error ?? false);
  if (continueOnError) {
    return { ...action, continue_on_error: true };
  }
  const result = { ...action };
  delete result.continue_on_error;
  return result;
}

describe("continue_on_error toggle", () => {
  it("should enable continue_on_error when currently undefined", () => {
    const action: NonConditionAction = {
      action: "light.turn_on",
      target: { entity_id: "light.bedroom" },
    };

    const result = toggleContinueOnError(action);

    expect(result.continue_on_error).toBe(true);
    expect(result.action).toBe("light.turn_on");
  });

  it("should enable continue_on_error when currently false", () => {
    const action: NonConditionAction = {
      action: "light.turn_on",
      continue_on_error: false,
    };

    const result = toggleContinueOnError(action);

    expect(result.continue_on_error).toBe(true);
  });

  it("should remove continue_on_error when currently true", () => {
    const action: NonConditionAction = {
      action: "light.turn_on",
      target: { entity_id: "light.bedroom" },
      continue_on_error: true,
    };

    const result = toggleContinueOnError(action);

    expect(result.continue_on_error).toBeUndefined();
    expect(result.action).toBe("light.turn_on");
    expect(result.target).toEqual({ entity_id: "light.bedroom" });
  });

  it("should preserve other action properties when toggling on", () => {
    const action: NonConditionAction = {
      alias: "Turn on bedroom light",
      action: "light.turn_on",
      target: { entity_id: "light.bedroom" },
      data: { brightness: 255 },
      enabled: true,
    };

    const result = toggleContinueOnError(action);

    expect(result.continue_on_error).toBe(true);
    expect(result.alias).toBe("Turn on bedroom light");
    expect(result.action).toBe("light.turn_on");
    expect(result.target).toEqual({ entity_id: "light.bedroom" });
    expect(result.data).toEqual({ brightness: 255 });
    expect(result.enabled).toBe(true);
  });

  it("should preserve other action properties when toggling off", () => {
    const action: NonConditionAction = {
      alias: "Turn on bedroom light",
      action: "light.turn_on",
      target: { entity_id: "light.bedroom" },
      continue_on_error: true,
      enabled: false,
    };

    const result = toggleContinueOnError(action);

    expect(result.continue_on_error).toBeUndefined();
    expect(result.alias).toBe("Turn on bedroom light");
    expect(result.enabled).toBe(false);
  });

  it("should work with delay action", () => {
    const action: NonConditionAction = {
      delay: "00:00:05",
    };

    const result = toggleContinueOnError(action);

    expect(result.continue_on_error).toBe(true);
    expect((result as any).delay).toBe("00:00:05");
  });

  it("should work with event action", () => {
    const action: NonConditionAction = {
      event: "custom_event",
      event_data: { key: "value" },
    };

    const result = toggleContinueOnError(action);

    expect(result.continue_on_error).toBe(true);
    expect((result as any).event).toBe("custom_event");
  });
});
```

#### 3.2 Create Test Directory

**Directory**: `/Users/abdul/Downloads/Projects/frontend/test/panels/config/automation/action/`

This directory doesn't exist yet, so it needs to be created.

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `cd /Users/abdul/Downloads/Projects/frontend && npm run test`
- [ ] All 7 test cases pass

#### Test Coverage:
- [ ] Toggle from undefined to true
- [ ] Toggle from false to true
- [ ] Toggle from true to undefined (removed)
- [ ] Preserve other properties when enabling
- [ ] Preserve other properties when disabling
- [ ] Works with different action types (delay, event)

**Implementation Note**: After completing this phase and all tests pass, proceed to Phase 4.

---

## Phase 4: Verify Translations

### Overview
The translation string already exists. This phase verifies it works correctly and adds a description tooltip if beneficial.

### Changes Required:

#### 3.1 Verify Existing Translation

**File**: `/Users/abdul/Downloads/Projects/frontend/src/translations/en.json`

**Location**: Line 4470

**Existing**: `"continue_on_error": "Continue on error"`

This string is already used for the tooltip on the icon indicator. The same string will be used for the menu item label.

#### 3.2 Optional: Add Tooltip Description

If user testing reveals confusion about what "Continue on error" means, we can add a description. However, the existing tooltip on the indicator icon provides the same label, so this may not be needed.

### Success Criteria:

#### Manual Verification:
- [ ] Menu item label displays "Continue on error" correctly
- [ ] No missing translation warnings in console

---

## Phase 5: Documentation

### Overview
Documentation for Home Assistant is maintained in a separate repository (`home-assistant/home-assistant.io`). This phase documents what needs to be updated there.

### Documentation Updates Needed

The automation documentation should mention that users can toggle "Continue on error" from the visual editor menu. The relevant docs are likely in:

- `source/_docs/automation/action.markdown` - Main action documentation
- Any visual editor tutorials

**Suggested addition** to action documentation:

> ### Continue on Error
>
> By default, if an action fails, the automation will stop executing subsequent actions. You can change this behavior per-action by enabling "Continue on error" from the action's menu in the visual editor, or by adding `continue_on_error: true` in YAML.
>
> Note: Only runtime errors (like a device being unavailable) can be continued from. Configuration errors (like referencing a non-existent service) will always stop the automation.

### Success Criteria:

#### Manual Verification:
- [ ] Note the documentation changes needed for home-assistant.io PR

---

## Testing Strategy

### Manual Testing Steps:

1. **Basic Toggle Test**:
   - Create new automation with a service call action
   - Open action menu (three-dot button)
   - Verify "Continue on error" menu item appears
   - Click to enable - verify icon changes to filled
   - Click again to disable - verify icon changes to outline
   - Save automation
   - Check YAML mode shows `continue_on_error: true` when enabled

2. **Persistence Test**:
   - Enable "Continue on error" on an action
   - Save and close automation
   - Reopen automation
   - Verify the filled icon shows in menu and header indicator appears

3. **Sidebar Test**:
   - Open action in sidebar (if available in your layout)
   - Verify menu item appears in sidebar menu
   - Verify toggling works from sidebar

4. **Condition Action Test**:
   - Add a condition action to automation
   - Open its menu
   - Verify "Continue on error" menu item does NOT appear (conditions don't support this)

5. **Multiple Actions Test**:
   - Create automation with 3 actions
   - Enable "Continue on error" on middle action only
   - Verify only that action shows the indicator icon
   - Save and verify YAML is correct

---

## References

- Research document: `thoughts/shared/research/2025-11-25-continue-on-error-visual-editor.md`
- Backend implementation: `homeassistant/helpers/script.py:491-579` (core repo)
- Backend schema: `homeassistant/helpers/config_validation.py:1463-1467` (core repo)
- Similar implementation (enabled toggle): `ha-automation-action-row.ts:430-446, 598-610`
