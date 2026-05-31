# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the project

Open `index.html` directly in a browser — no build step, no server, no dependencies. Just double-click the file or use a browser's "Open file" dialog.

To run the inline tests, open the browser console after loading the page. `runTests()` is called automatically on every load and logs `console.assert` failures if any exist.

## Architecture

The entire application lives in a single `index.html` file: CSS in `<style>`, JavaScript in `<script>`, no external dependencies.

**State → render loop.** All application state lives in one `state` object. Every user action mutates `state`, calls `save()` to persist to `localStorage`, then calls `render()` which rebuilds the entire `#app` innerHTML from scratch using template literals. There is no virtual DOM, diffing, or framework.

**Persistence.** Three `localStorage` keys:
- `raz_learning_progress_dashboard_bilingual_v1` — the tracks array (JSON)
- `raz_learning_progress_dashboard_bilingual_v1_checkboxes` — weekly checkbox state (JSON)
- `raz_learning_progress_dashboard_bilingual_v1_language` — active language (`"en"` or `"he"`)

On first load with no saved data, `defaultTracks` is deep-cloned via `structuredClone()` as the initial state.

## Data model

**Track:** `{ id, name, layer, description, items[] }`

**Item:** `{ id, name, score (1–10), status, next, tag }`

**Valid layers** (used in filter UI and new-track form): `"Core Stack"`, `"Skills"`, `"Expansion"`, `"Business"`. `"All"` is a UI-only filter option, never stored on a track.

**Valid item tags:** `"Active"`, `"Important"`, `"Learning"`, `"Explore"`, `"Later"`, `"Strong"`

**Score tiers** (`toneClass` / `scoreLabel`):
- 1–2 → `tone-start` / "Start"
- 3–4 → `tone-learning` / "Learning"
- 5–6 → `tone-working` / "Working"
- 7–8 → `tone-strong` / "Strong"
- 9–10 → `tone-mastery` / "Mastery"

Track score is computed in `calculateTrackScore()` as the rounded average of its items' scores (clamped 1–10).

## State object

```js
state = {
  tracks: [],          // array of track objects (the persisted data)
  query: "",           // current search string
  layer: "All",        // active layer filter
  expanded: null,      // id of the single open track (accordion — only one at a time)
  editing: null,       // { trackId, itemId } of the item currently in edit mode, or null
  newTrackOpen: false, // whether the add-track form is visible
  newTrack: { name, layer, description }, // form buffer for new track
  checkboxes: {},      // { week1: bool, week2: bool, week3: bool }
  language: "en"       // "en" | "he"
}
```

The weekly checklist has **three hardcoded items** (`week1`, `week2`, `week3`) whose text comes from `i18n.check1/2/3`. It is not dynamically generated from `state.tracks`.

New tracks are prepended (`unshift`) to `state.tracks`. New items are also prepended to `track.items`.

## JSON backup format

`exportJson()` produces:

```json
{
  "exportedAt": "<ISO timestamp>",
  "language": "en",
  "tracks": [ /* full tracks array */ ],
  "checkboxes": { "week1": true, "week2": false, "week3": false }
}
```

`importJson()` accepts **either** this full export format **or** a bare tracks array (for backward compatibility). It reads `parsed.tracks` if present, otherwise treats the root as the array. `language` and `checkboxes` are optional in the import; missing fields default to `"en"` / `{}`.

## i18n

All UI strings are in the `i18n` object with `en` and `he` keys. The `t(key)` helper reads from `state.language`. Language also sets `<html dir>` (ltr/rtl) and `<html lang>` via `applyLanguage()`, which is called at both startup and on every `render()`.

Tech terms (tool/library names) are wrapped with `tech(value)` — an `.ltr` span — so they always render left-to-right even in RTL (Hebrew) mode.

## XSS safety

All user-supplied and i18n strings use `esc()` (HTML entity escaping) before insertion into template literals. `tech()` also calls `esc()` internally. Inline event handlers use `esc()` on any attribute values they embed.

## Inline tests

`runTests()` at the bottom of the script uses `console.assert` to test: `clampScore`, `scoreLabel`, `calculateTrackScore`, `makeItem`, and the `i18n` direction/translation fields. Add new assertions directly to `runTests()` — no test runner or framework is involved.
