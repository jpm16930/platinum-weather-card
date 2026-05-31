# GUI Editor Issues — ha-select Dropdown Not Functional in HA 2026.5

## Background

The platinum-weather-card editor (`src/editor.ts`) uses `ha-select` + `mwc-list-item` children
for all dropdown controls (slot selects, layout options, time format, unit system, etc.).
In HA 2026.3+, `ha-select` was migrated to a new WebAwesome-based implementation wrapping
`ha-dropdown`. This broke all dropdown event handling.

The editor renders correctly and dropdowns open, but selecting an item does not update the
config. The GUI editor goal — "all card config settable without touching YAML" — is unfinished
for selects.

---

## What Was Tried (and Failed)

### 1. `@selected` event with `ev.target.value`
**Original code.** `ha-select` previously fired `selected` in older HA versions.
In HA 2026.5 this event no longer fires (confirmed by comprehensive event listener test).

### 2. `@value-changed` event
Changed all `@selected` → `@value-changed`. HA's `ha-entity-picker` uses `@value-changed`
successfully, so it seemed correct. However, ha-select does NOT fire `value-changed` in 2026.5.
Confirmed by diagnostic: console listener attached directly to the ha-select element caught zero
`value-changed` events.

### 3. `@click` on ha-select via LitElement template
LitElement template event bindings (`@click=${handler}`) on ha-select never fire.
The console.log in the handler was never reached, even though a manually-added
`sel.addEventListener('click', ...)` on the same element DID fire.
Root cause unknown — possibly related to LitElement's EventPart mechanism interacting
badly with the new WebAwesome-based ha-select.

### 4. Programmatic `addEventListener('click', ...)` in `updated()` lifecycle
Using `this.renderRoot.querySelectorAll('ha-select').forEach(el => el.addEventListener('click', handler))`
in `updated()`. This catches the "open dropdown" click (fires with current value = no change),
but NOT the "select item" click.

Likely explanation: ha-select's dropdown items are rendered in a portal or the ha-select
element is replaced by a LitElement re-render between the open-click and select-click,
causing the listener to be on the wrong element instance.

### 5. Document-level `click` listener polling ha-select values
`document.addEventListener('click', () => { check all ha-select values... })` fires after
every click document-wide, then reads `el.value` on each ha-select to detect changes.

Result: `el.value` always showed the old (config) value even after the user clicked "Imperial".
ha-select is a **controlled component** — it temporarily holds the new value during the click
event propagation, then reverts to the config-bound value. The document listener fires too
late (after the revert).

### 6. Reading `ev.target.value` during event propagation (console test)
A manually-added `sel.addEventListener('click', ...)` in the browser console DID correctly
show `value=imperial` during the click event. This is the only mechanism confirmed to work.
But attempts to replicate this via `updated()` only caught the open-click (value=current),
not the item-selection click.

---

## Diagnostic Evidence

- HA 2026.5.4, platinum-weather-card 1.0.5
- Comprehensive event test on ha-select element (all common event names):
  `['selected','value-changed','change','input','sl-change','wa-change','action','closed','click']`
  → **Only `click` fired**, and only for the "open dropdown" click (not item selection)
- Console `sel.addEventListener('click', ...)` DID fire for item selection with correct value
- LitElement `@click=${handler}` template binding on the same element — never fired

---

## Known Working Alternative

Community post reference:
https://community.home-assistant.io/t/custom-cards-with-gui-editor-as-of-2023/542254

Post #14 shows a working approach:
- Extend plain `HTMLElement` (not LitElement)
- Create ha-select via `document.createElement('ha-select')`
- Append `mwc-list-item` children programmatically
- Use `addEventListener('change', ...)` — fires correctly on programmatically-created ha-select

The `change` event fires on ha-select when created via `document.createElement` but NOT
when created via LitElement's HTML template rendering. The reason for this difference
is not fully understood.

## Potential Future Approaches

1. **Rewrite editor to extend `HTMLElement`** and create all UI elements programmatically
   (like the community post). Matches what works but is a large rewrite (~1700 lines).

2. **Replace `ha-select` + `mwc-list-item` with `ha-selector`** (select type).
   `ha-selector` is HA's officially maintained selector component. Entity pickers
   (`ha-entity-picker`) already work via `@value-changed`. `ha-selector` with select schema
   should fire `value-changed` correctly. Requires replacing all ha-select usages and
   defining option arrays for each dropdown.

3. **Monitor HA frontend updates** for a re-stabilised ha-select event API.

---

## Current State (as of 2026-05-31)

- GUI editor renders correctly, dropdowns open and show options
- Selecting an item has no effect (config not updated)
- Entity pickers (ha-entity-picker) work correctly via `@value-changed`
- Text fields (ha-textfield) work correctly via `@input`  
- Toggles (ha-switch) work correctly via `@change`
- Only ha-select dropdowns are broken
- The unit_system (metric/imperial) override must be set via YAML: `unit_system: imperial`
