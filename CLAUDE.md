# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A hardware-free demo of a smart-home system for a school project, built as standalone HTML files with inline CSS/JS — no build system, no dependencies, no tests. The app simulates the home controller entirely in the browser. All UI text is Hebrew, and the document is RTL (`<html lang="he" dir="rtl">`).

- `index.html` — the current app. All work happens here.
- `smart-home-demo.html` — the first, superseded version (dark theme). Kept for comparison only; don't extend it.
- `manifest.webmanifest`, `sw.js`, `icons/` — PWA layer so the app installs to an iPhone home screen ("Add to Home Screen" from Safari) and works offline. The service worker only registers over https/localhost, so local file:// development is unaffected. When changing cached assets, bump the `CACHE` version in `sw.js`.

## Running

Open `index.html` directly in a browser (double-click, or the Claude browser preview). No server needed. State lives in memory only — refreshing resets the demo to its initial state, which is intentional for presentations; do not add persistence unless asked.

The Heebo font loads from Google Fonts; offline it silently falls back to system fonts.

## Architecture of index.html

Single file: CSS in `<style>`, markup skeleton, then one `<script>` with clearly commented sections (ICONS, TYPES, STATE, HEADER, ROOM CHIPS, SCENE, DEVICE CARDS, ACTIONS, PRESETS, SCHEDULES, LOG & TOASTS, SHEETS, TABS, INIT).

### State

Everything lives in the global `S`, produced by `initialState()` (which is also the demo-reset source):

- `S.rooms[]` — `{id, name, loc, devices[]}`; a device is `{id, type, name, on}` (+ `onSince`/`alerted` for outlets). `type` is a key of `TYPES` (lamp, shutter, outlet, ac, tv, boiler, lock, ir). For locks, `on === locked` (locked is the "active/secure" state); shutters, `on === open`.
- `S.presets[]` — `{id, name, icon, actions:[{deviceId, on}]}`. A preset's live status (`presetStatus`) is derived by comparing actions to current device state: all match → `match`, none → `none`, else `partial`.
- `S.schedules[]` — `{id, time:'HH:MM', name, enabled, actions:[{deviceId, on}]}`. A 10s interval fires enabled schedules when the wall clock hits their time (`_fired` stamp prevents re-firing within the minute).
- `uid` counter for all new ids; `editMode` gates destructive UI.

### Rendering conventions

DOM is rebuilt with `innerHTML` per section (`renderChips`, `renderCards`, `renderSchedule`, `renderPresets`...). The one exception is the room scene: `buildScene()` constructs it, but state changes go through `updateSceneState()`, which only toggles classes — this is what preserves CSS transitions (shutter slide, lamp glow, room dim). Rebuild the scene only on structural changes (room switch, add/remove device); toggle classes for state changes.

`renderHeader()` doubles as the universal "state changed" hook: it is called after every mutation and once per second by the clock tick, and it piggybacks `renderPresets()` so preset status dots stay live. After mutating device state, call `afterChange()` (scene classes + cards + header) rather than individual renderers.

Scene visuals are per-type absolutely-positioned elements defined by `ANCHORS`/`sceneEl`; multiple devices of a type offset by index. Adding a new device type means touching: `TYPES`, `I` (icon), `ANCHORS`, `sceneEl` + its CSS block, and `actionVerbs`/`verbFor` if its on/off verbs differ (like shutter/lock).

### UI patterns

- Bottom sheet (`openSheet(html)`/`closeSheet`) is the single modal mechanism — demo menu, add/edit device, room, preset, and schedule forms all render into `#sheetBody`.
- `editMode` (pencil button in the header, next to the demo-menu flask) is the only path to destructive actions: chip pencils → room edit sheet, ✕ on device/preset cards, add buttons. Room deletion uses a two-click confirm on the same button.
- Events log via `logEvent(text, kind)` with kinds `app` (user command), `sensor` (simulated physical change), `ctrl` (controller/schedule/scenario), `alert` — the kind drives the log dot color and is part of the demo's story (app displays true physical state, logic lives in the controller, not the phone).
- The demo menu (flask icon) simulates controller-initiated events for presentations; demo actions must go through the same state + log + toast paths as real ones.
