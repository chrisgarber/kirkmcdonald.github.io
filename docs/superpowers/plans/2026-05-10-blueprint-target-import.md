# Blueprint Target Import Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Blueprint tab that imports a Factorio blueprint string as normal calculator item-rate targets.

**Architecture:** Add a dependency-light `blueprint.js` module for decoding, counting, applying counts to targets, and rendering import status. Wire it through `events.js` and `calc.html` using the app's existing global `handlers` pattern. Keep tests focused on pure helpers and a small fake `spec`.

**Tech Stack:** Static ES modules, browser DOM APIs, existing global `pako`, Node `node:test` for helper tests.

---

### Task 1: Blueprint Helper Tests

**Files:**
- Create: `tests/blueprint.test.mjs`
- Create later: `blueprint.js`

- [ ] **Step 1: Write the failing test**

```javascript
import assert from "node:assert/strict"
import test from "node:test"
import { deflateSync, inflateSync } from "node:zlib"

import {
    applyBlueprintCounts,
    countBlueprintItems,
    decodeBlueprintString,
    getBlueprintRoot,
} from "../blueprint.js"

function encodeBlueprint(payload) {
    return "0" + deflateSync(JSON.stringify(payload)).toString("base64")
}

function inflate(bytes) {
    return inflateSync(Buffer.from(bytes)).toString("utf8")
}
```

Use real tests for:
- decoding a minimal blueprint string;
- rejecting blueprint books;
- counting entities and tiles according to options;
- applying counts by replacing existing targets and setting target rates;
- preserving known imports while reporting unknown item names.

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test tests/blueprint.test.mjs`

Expected: FAIL with `Cannot find module '../blueprint.js'`.

### Task 2: Blueprint Helper Implementation

**Files:**
- Create: `blueprint.js`
- Test: `tests/blueprint.test.mjs`

- [ ] **Step 1: Implement minimal helpers**

Add:

```javascript
export function decodeBlueprintString(value, inflate = defaultInflate) { ... }
export function getBlueprintRoot(decoded) { ... }
export function countBlueprintItems(root, options) { ... }
export function applyBlueprintCounts(spec, counts) { ... }
```

Key behavior:
- Require strings to start with Factorio blueprint version byte `0`.
- Base64-decode the rest with `atob` in browsers and `Buffer` in Node.
- Inflate with injected inflate function or `globalThis.pako.inflate`.
- Count `root.entities[].name` when `options.entities` is true.
- Count `root.tiles[].name` when `options.tiles` is true.
- Sort imported target rows deterministically by item metadata.
- Remove current targets before creating imported targets.
- Set each target rate with the count string, not the building count.

- [ ] **Step 2: Run test to verify it passes**

Run: `node --test tests/blueprint.test.mjs`

Expected: PASS.

### Task 3: Blueprint UI Wiring

**Files:**
- Modify: `events.js`
- Modify: `calc.html`
- Modify: `calc.css`
- Modify: `blueprint.js`

- [ ] **Step 1: Add DOM import wrapper**

In `blueprint.js`, add:

```javascript
export function importBlueprintFromDocument(spec, doc = document) { ... }
```

This reads:
- `#blueprint_string`
- `#blueprint_include_entities`
- `#blueprint_include_tiles`
- `#blueprint_status`

It decodes, counts, applies counts, calls `spec.updateSolution()`, and renders success or error messages.

- [ ] **Step 2: Wire event handler**

In `events.js`, import the wrapper and export:

```javascript
export function importBlueprint() {
    importBlueprintFromDocument(spec)
}
```

In `calc.html`, include `importBlueprint` in the module import and assign `handlers.importBlueprint = importBlueprint`.

- [ ] **Step 3: Add the tab markup**

Add a Blueprint tab button next to Factory:

```html
<button class="tab_button" id="blueprint_button" onclick="handlers.clickTab('blueprint')">Blueprint</button>
```

Add `#blueprint_tab` with textarea, entity/tile checkboxes, import button, and status div.

- [ ] **Step 4: Add small CSS**

Style `#blueprint_string`, `.blueprint-options`, `.blueprint-status`, `.blueprint-error`, and `.blueprint-warning` with existing colors and form conventions.

### Task 4: Verification

**Files:**
- All changed files.

- [ ] **Step 1: Run automated tests**

Run: `node --test tests/blueprint.test.mjs`

Expected: PASS.

- [ ] **Step 2: Run a static server**

Run: `python3 -m http.server 8000`

Expected: Server starts and serves the static app.

- [ ] **Step 3: Browser smoke test**

Open `http://localhost:8000/calc.html`, switch to Blueprint, paste a minimal encoded blueprint, click Import, and confirm the Factory targets update.

- [ ] **Step 4: Commit**

Run:

```bash
git add blueprint.js events.js calc.html calc.css tests/blueprint.test.mjs docs/superpowers/plans/2026-05-10-blueprint-target-import.md
git commit -m "Add blueprint target import"
```
