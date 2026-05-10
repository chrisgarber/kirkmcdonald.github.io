# Blueprint Target Import Design

## Goal

Add a Factorio blueprint import feature that turns a blueprint string into normal calculator targets. The first version counts blueprint entities and tiles, then uses the calculator's active data set and settings to calculate upstream requirements.

## User Flow

1. The user selects the calculator data set and recipe settings as usual.
2. The user opens a new Blueprint tab.
3. The user pastes a Factorio blueprint string.
4. The user chooses import options:
   - Entities: enabled by default.
   - Tiles: enabled by default.
5. The user clicks Import.
6. The calculator replaces the current Factory targets with the counted blueprint items as item-rate targets and runs the normal solution.

## Scope

Version 1 includes:

- Single blueprint strings.
- Entity counts from `blueprint.entities[].name`.
- Tile counts from `blueprint.tiles[].name`.
- Import-time options for entities and tiles.
- Unknown-item reporting in the Blueprint tab.
- Normal Factory target creation for every known counted item.

Version 1 excludes, but leaves room for:

- Blueprint books.
- Modules installed in entities.
- Logistic requests and filters.
- Fuel requests.
- Train schedules.
- Entity recipe configuration.
- Quality-specific item handling.
- A separate standalone raw-material report.

## Architecture

Add a focused `blueprint.js` module with pure helpers and one DOM-facing import function.

- `decodeBlueprintString(value)`: validates the leading blueprint version byte, base64-decodes the body, inflates it with the existing `pako` dependency, and parses JSON.
- `getBlueprintRoot(decoded)`: accepts only a single `blueprint` root in version 1 and reports other roots as unsupported.
- `countBlueprintItems(root, options)`: returns a map of item key to count from enabled blueprint sections.
- `applyBlueprintCounts(counts)`: replaces existing targets and creates one item-rate target per known item.

Wire this through `events.js` with an `importBlueprint` handler exposed from `calc.html`, following the existing global `handlers` pattern.

## Data Flow

Blueprint import uses current calculator state instead of hard-coding Space Age:

```text
blueprint string
  -> decode/decompress JSON
  -> count entity and tile names
  -> match names against spec.items
  -> replace build targets
  -> spec.updateSolution()
```

Each created target uses the existing target model with its item rate set to the blueprint count for the current display interval. This is equivalent to asking the calculator for the production graph needed to build one blueprint batch per displayed second, minute, or hour. The default display interval is per minute, so imported target values read as blueprint item counts in the normal default view. This keeps all downstream calculation behavior inside the current solver, recipe selection, planet filtering, disabled recipes, priorities, modules, display formatting, and visualization.

## Error Handling

The Blueprint tab shows actionable messages for:

- Empty input.
- Invalid blueprint string format.
- Invalid base64, zlib, or JSON content.
- Unsupported blueprint-book roots.
- No entities or tiles selected.
- No matching items found.

Unknown entity or tile names are skipped and listed after import. Known items are still imported so one unsupported item does not block the rest of the blueprint.

## UI

Add a Blueprint tab near the existing Factory tab. The tab contains:

- A textarea for the blueprint string.
- Checkboxes for Entities and Tiles.
- An Import button.
- A compact status/results area for imported counts, skipped names, and errors.

The UI should use the project's existing plain table/form styling rather than introducing a new visual system.

## Tests

Add focused tests for:

- Decoding a minimal valid blueprint string.
- Counting entities and tiles based on options.
- Reporting unsupported blueprint books.
- Skipping unknown item keys while importing known ones.
- Replacing existing targets with imported targets.

Because the app is static ES modules, tests can run in Node for pure helpers and use small fakes for `spec` and target behavior where DOM interaction is not required.
