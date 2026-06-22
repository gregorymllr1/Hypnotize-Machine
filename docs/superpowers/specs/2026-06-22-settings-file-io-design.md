# Settings File Import/Export — Design

**Date:** 2026-06-22
**Component:** `Hypnotize Machine.html` (single-file vanilla-JS Web Audio drum machine)
**Goal:** Make the machine's settings portable — export the full current state to a downloadable `.json` file and import one back — and round out the saved schema so it captures everything (tape, sidechain, master, polymeter lane lengths). This is **piece 1 of 3** in the larger "song ingestion" feature; it builds the settings-file container that the audio-analysis engine (piece 2) and interpretation layer (piece 3) will later fill.

---

## 1. Decisions (locked during brainstorming)

| # | Decision | Choice |
|---|----------|--------|
| 1 | Build order | Decompose into 3 pieces, build 1→2→3. This spec is **piece 1** only. |
| 2 | Audio in export | **Settings only** — do NOT embed sample/chop audio bytes (parity with current localStorage behavior). Self-contained audio bundling is deferred to piece 3. |
| 3 | Import behavior | Import **applies to the session AND stores into localStorage** + refreshes the Load dropdown, so imported files double as a retrievable library. |

---

## 2. Existing state (what's there today)

- Settings persist to `localStorage` key `hypnotize_v2` as:
  `{version:2, bpm, swing, patterns:[{name, lanes:[{...}]}], songSeq, songMode}`.
- `saveCurrentPattern()` serializes; `loadPattern(name)` restores. Loaders handle `version:1` and `version:2`.
- **Gaps in the saved schema** (captured nowhere, so lost on save/load):
  - `master` (master volume, from `volEl.value`)
  - `tape` (`tapeAmt` macro)
  - `sidechain` (`scAmt` macro)
  - per-lane `length` (polymeter; lanes are serialized with an explicit field list that omits `length`)
- Per-step p-locks (`decay`, `pan`, `filter`, `note`, `glide`, `slice`, `_timingOffset`) already round-trip because steps are serialized via `{...s}` (whole-object spread).
- Audio buffers are intentionally NOT persisted; sample/chop lanes store `fname` and reload audio from a file picker.

---

## 3. Schema completion (bump to `version:3`)

The exported/saved object becomes:

```js
{
  meta: { app:'Hypnotize Machine', schema:3, title:'<string>', created:'<ISO 8601>' },
  version: 3,
  bpm, swing,
  master,        // NEW — parseFloat(volEl.value)
  tape,          // NEW — tapeAmt
  sidechain,     // NEW — scAmt
  patterns: [
    { name, lanes: [ { id, name, type, steps:[{...s}], gain, mute, tune, decay,
                       semis, fname, slices, warp, length } ] }   // length NEW
  ],
  songSeq, songMode
}
```

- `meta` is informational; loaders ignore unknown fields.
- Back-compat: `version:1` and `version:2` loaders stay intact. When applying any version, missing `master`/`tape`/`sidechain` default to the current control values (or 0.9 / 0 / 0), and missing per-lane `length` defaults to `16`.

---

## 4. Refactor into a clean seam

Two functions become the single source of truth (this is the seam pieces 2 & 3 target):

### `buildSettings(title)` → object
Reads current globals (`bpm`, `swing`, `tapeAmt`, `scAmt`, `volEl.value`, `patterns`, `songSeq`, `songMode`) and returns the `version:3` object in §3. Per-lane serialization adds `length`; the `meta` block is stamped with `title`, `created:new Date().toISOString()`, `app`, `schema:3`.

### `applySettings(obj)` → void
Applies a settings object to the machine:
- Set `bpm`, `swing` and their UI labels/inputs (as `loadPattern` does today).
- Set `master` → `volEl.value` + `master.gain` (if audio inited) + label; `tape` → `tapeAmt` + `tapeEl.value` + label + `updateTape()` (if inited); `sidechain` → `scAmt` + `scEl.value` + label.
- Rebuild `patterns` from `obj.patterns` (version-aware: reuse existing v2/v1 branches), mapping steps through `coerceStep`, setting `buffer:null`, normalizing chop `slices`/`warp`, and restoring per-lane `length` (default 16).
- Set `curPat=0`, `lanes=patterns[0].lanes`, `songSeq`, `songMode`.
- Recompute `smpCounter`/`chopCounter`/`bassCounter`.
- Refresh UI: `rebuildGrid()`, `buildPatBtns()`, `buildSongSlots()`, song-toggle state, `updatePresetDropdown()`.

### Wiring
- `saveCurrentPattern()` → `const o=buildSettings(name); saves[name]=o; …` (store to localStorage, unchanged UX).
- `loadPattern(name)` → `const o=saves[name]; applySettings(o);`.

No behavior change to existing localStorage saves — just deduplicated, with the wider schema.

---

## 5. Export / Import UI

Two buttons added to the header transport, next to **Save** / **Load…** / **delete**:

### Export (`#exportSettings`)
1. `const currentTitle = document.getElementById('loadPreset').value || 'untitled';`
   `const title = (prompt('Settings file name:', currentTitle) || 'untitled').trim();`
   (defaults the prompt to the currently-selected Load-dropdown preset name, else "untitled".)
2. `const obj = buildSettings(title)`.
3. `downloadBlob(new Blob([JSON.stringify(obj,null,2)], {type:'application/json'}), sanitize(title)+'.hypnotize.json')`.
   - `sanitize` strips characters invalid in filenames → `[^a-z0-9-_]` to `-`, lowercased.
   - `downloadBlob` already exists and is reused.

### Import (`#importSettings` + hidden `<input type=file accept=".json,application/json">`)
1. Button click → `file.click()`.
2. On change → `file.text()` → `JSON.parse` in `try/catch`.
3. Validate: result is a non-null object with either `patterns` (v2/v3) or `lanes` (v1). Otherwise `alert('Not a valid Hypnotize settings file')` and abort.
4. `applySettings(obj)`.
5. Persist: derive a name (`obj.meta?.title` || file basename), store into localStorage saves under that name, `updatePresetDropdown()`, select it. This makes the imported file retrievable later via the Load dropdown.

---

## 6. Error handling

- `JSON.parse` failure or non-object → caught, `alert`, no state mutation.
- Missing/partial fields → `applySettings` defaults them (`coerceStep`, `||` fallbacks, `length:16`), so a sparse but structurally valid file still loads.
- Importing a file whose sample/chop lanes reference audio: lanes load with `buffer:null` and the stored `fname` shown — identical to localStorage today. User reloads audio via the lane's file picker.
- localStorage quota errors on import-save are caught and surfaced via `alert`, but the session still applies the settings.

---

## 7. Testing / QA plan

**Manual QA** (consistent with the project — single HTML file, no test harness), run via `python -m http.server 8765`:

1. Build a beat that exercises the new schema: change BPM/swing, set master/tape/sidechain knobs, set a polymeter lane `Len` to something ≠ 16, add per-step p-locks (pan/filter/decay) and a humanized lane.
2. **Export** → a `.json` downloads; inspect it contains `meta`, `version:3`, `master`/`tape`/`sidechain`, and per-lane `length`.
3. Reload the page (fresh defaults), **Import** the file → assert restored: bpm, swing, master/tape/sidechain knob positions, the polymeter lane length, p-lock values, song sequence.
4. Confirm the imported file now appears in the **Load** dropdown and re-loads from there.
5. Import a deliberately malformed `.json` and a random non-settings `.json` → both `alert` cleanly with no state change.
6. Load an old `version:2` localStorage save → still loads (back-compat).

**Round-trip check** (verifiable in headless browser): `applySettings(buildSettings('t'))` is a no-op on observable state; export→parse→apply yields matching `bpm`, `master`, `tape`, `sidechain`, lane `length`s, and step fields.

---

## 8. Out of scope (deferred)

- Embedding sample/chop audio in the file (piece 3 — extemporaneous samples).
- Any audio analysis or auto-generation (pieces 2 & 3).
- Cloud/remote storage of settings files (local download/upload only).
- Migration UI or settings-file versioning beyond load-time back-compat.
