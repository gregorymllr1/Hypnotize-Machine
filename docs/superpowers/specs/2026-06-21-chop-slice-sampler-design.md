# Chop / Slice Sampler — Design

**Date:** 2026-06-21
**Component:** `Hypnotize Machine.html` (single-file vanilla-JS Web Audio drum machine)
**Goal:** Add an SP-1200-style chop/slice sampler: load a loop, place slice markers (manual or auto via transient/grid detection), map slices to steps, and lock them to the grid — either by gating/repitching or by pitch-preserved granular time-stretch.

---

## 1. Decisions (locked during brainstorming)

| # | Decision | Choice |
|---|----------|--------|
| 1 | Integration model | **New `'chop'` lane type** (reuses patterns, song mode, swing, lane gain/mute, export, saves) |
| 2 | Tempo lock | **Both warp modes, switchable per lane**: `gate` (default, authentic) + `stretch` (granular, pitch-preserved) |
| 3 | Slice → step mapping | **Auto-fill left-to-right + per-step override** via the existing right-click popover |
| 4 | Auto-slice modes | **Both** transient detection (sensitivity slider) and equal grid divisions (4/8/16/32) |
| 5 | Gate window | **Until next active step** in the lane (whole bar if none) — allows holding/stretching a slice across steps |
| 6 | Default kit | **Seed one empty chop lane** in `defaultLanes()` for discoverability |
| 7 | Testing | **Manual QA only** (browser + local `python -m http.server`). Pure logic still factored into standalone functions for clarity, but no automated harness is added. |

---

## 2. Data model

New lane `type:'chop'`, extending the existing lane object shape:

```js
{
  id:'chop1', name:'Chop 1', type:'chop',
  gain:0.85, mute:false,
  buffer:null,        // decoded AudioBuffer of the loop (NOT persisted — same as samples)
  fname:null,
  slices:[0],         // sorted normalized start points in [0,1); slice i spans slices[i]..(slices[i+1] || 1)
  warp:'gate',        // 'gate' | 'stretch'
  semis:0,            // optional pitch offset, reuses the sample-lane concept
  steps:[ … ]         // 16 steps
}
```

**Step model change:** the shared step gains one field — `slice` (integer index into `slices`, default `-1` = unassigned).

- `makeStep(roll=0)` → `{roll, vel:1.0, prob:1.0, glide:1, note:36, slice:-1}`
- `coerceStep(s)` carries `slice` through (default `-1`) for backward-compatible loads.
- The new field is inert for non-chop lanes.

A chop step produces sound only when `roll > 0 && slice >= 0 && slice < L.slices.length`.

Number of slices for lane `L` = `L.slices.length` (each marker starts a slice; the final slice ends at `1.0`).

---

## 3. UI

### 3.1 Chop lane row (`laneRow` extensions for `type==='chop'`)

In the lane's `.extra` area:
- **Load loop** button + hidden `file` input + `fname` label (reuses `loadSample`-style decode).
- **Edit slices ▾** toggle — opens/closes the slice editor panel (§3.3), bound to this lane.
- **Warp toggle** — `[ Gate | Stretch ]`, sets `L.warp`.
- **Pitch** mini-range (`-12..+12`), sets `L.semis` (mirrors sample lane).

Step cells (`refreshCellVis` for chop lanes):
- When `roll>0 && slice>=0`: cell shows the **slice number** as its label (e.g. `3`). Rolls still annotate with `×N` (e.g. `3×2`) — slice number takes the primary glyph, roll count is secondary/superscript-style.
- When off: existing dot.
- Probability dimming/tilde behavior unchanged.

### 3.2 Step interaction

- **Left-click** cycles roll as today (`off→1→2→3→4→6→off`). When a step turns on (roll 0→1) and its `slice` is still `-1`, default it to `stepIdx % numSlices` (or `0` if no slices yet).
- **Right-click** opens the shared `#stepCtrl` popover, which gains a **Slice** row (a number stepper / select, bounded `0..numSlices-1`) shown only for chop-lane steps, above the existing Vel/Prob rows.

### 3.3 Slice editor panel

A single collapsible `<div id="chopEditor">` inserted **immediately after the `.scroll` grid container** (the 46px lane row is too short for a waveform). It binds to one chop lane at a time (`chopEditorLaneId`).

Contents:
- **Waveform `<canvas>`** (full width, ~120px tall): downsampled min/max peak render of channel 0. Vertical numbered lines = slice markers. A moving playhead line tracks the currently sounding slice during playback (best-effort, from the draw loop).
- **Marker editing on the canvas:**
  - Click empty area → add a marker at that normalized x.
  - Drag a marker → move it (clamped between neighbors).
  - Double-click a marker → delete it. Marker at `0` is implicit and not deletable.
- **Auto-slice controls:**
  - Mode select: `Transient | Grid`.
  - *Transient* → **Sensitivity** slider (maps to detection threshold).
  - *Grid* → **Divisions** select: `4 / 8 / 16 / 32`.
  - **Auto-slice** button → recomputes `slices`, then auto-runs **Map to steps**.
- **Map to steps** button → fills the lane's steps left-to-right: step `i` gets `roll=1, slice=i` for `i < min(STEPS, numSlices)`; remaining steps cleared. (Re-runnable.)
- **Clear slices** button → `slices=[0]`, clears slice assignments.
- Closing the panel (toggle off, lane deleted, or pattern switch removing the lane) hides it.

---

## 4. Transient detection (pure function)

`detectTransients(channelData, sampleRate, sensitivity) -> number[]` (normalized markers, includes `0`).

Pragmatic energy-based onset detection (no FFT):
1. Window the signal into ~10ms hops (`hop = round(sampleRate * 0.010)`).
2. For each hop compute RMS energy.
3. Onset detection function = `max(0, rms[i] - rms[i-1])` (positive flux).
4. Normalize the onset function; threshold = `1 - sensitivity` (higher sensitivity → lower threshold → more slices).
5. Peak-pick: a hop is an onset if its onset value is above threshold AND it is a local max AND it is at least `minSpacing` (~50ms) after the previous accepted onset.
6. Convert accepted hop indices → normalized sample positions. Always prepend `0`. Sort, dedupe.

Cap the number of detected slices (e.g. max 64) to keep the UI and mapping sane.

---

## 5. Playback engine

### 5.1 Gate window helper (pure)

`gateWindow(steps, stepIdx, stepDur) -> seconds`

Number of steps from `stepIdx` to the next active step (`roll>0 && slice>=0`), scanning forward with wraparound over the 16-step loop. If no other active step exists, the window is the whole bar (`STEPS * stepDur`). Returns `stepsUntilNext * stepDur`. (Swing is ignored in the window length for simplicity; trigger timing still respects swing.)

**Rolls:** a rolled step (`roll>1`) fires `roll` sub-hits spaced `stepDur/roll` apart within its own step. Each sub-hit's effective window is therefore `stepDur/roll`, NOT the full `gateWindow` — rolls subdivide within their step rather than holding across steps. So `chopVoice` selects its window as:

```
effWin = (L.steps[stepIdx].roll > 1) ? stepDur()/roll : gateWindow(L.steps, stepIdx, stepDur())
```

**Wiring note:** the live `scheduleStep` currently calls `trigger(L, tt, vel)` without a step index — it must be changed to `trigger(L, tt, vel, step)` so `chopVoice` (and the existing `bass808`) receive `stepIdx`. The same applies to the offline render loop (§7.1).

### 5.2 `chopVoice(t, v, L, stepIdx)` — added to the `trigger` switch (`case 'chop'`)

Common setup:
- `idx = L.steps[stepIdx].slice`; bail if `!L.buffer || idx<0 || idx>=L.slices.length`.
- `dur = L.buffer.duration`
- `sliceStart = L.slices[idx] * dur`
- `sliceEnd = (idx+1 < L.slices.length ? L.slices[idx+1] : 1) * dur`
- `sliceLen = sliceEnd - sliceStart`
- `rate = 2^(L.semis/12)`
- `win = (roll>1) ? stepDur()/roll : gateWindow(L.steps, stepIdx, stepDur())` (see §5.1)
- destination = `scBus || master` (so chops duck with the sidechain like samples).

**Gate mode (`L.warp==='gate'`):**
- One `BufferSource` (`buffer=L.buffer`, `playbackRate=rate`).
- Through a gain node: short attack (~2ms) and a short release fade (~5ms) at the gate end to prevent clicks; level = `max(v*L.gain, 0.0003)`.
- `src.start(t, sliceStart, Math.min(sliceLen, win * rate))` — play from the slice offset for either its natural length or the gate window (in source seconds), whichever is shorter.

**Stretch mode (`L.warp==='stretch'`) — granular, pitch-preserved:**
- Conform `sliceLen` of audio into `win` output seconds.
- `grainSize ≈ 0.08s`, `hop = grainSize / 2` (50% overlap), in **output** time.
- `nGrains = clamp(ceil(win / hop), 1, 200)`.
- For grain `k` at output offset `og = k*hop`:
  - source read position `sp = sliceStart + (og / win) * sliceLen`
  - one `BufferSource` (`playbackRate=rate`) through a per-grain gain node shaped as a **Hann window** (ramp up over `grainSize/2`, down over `grainSize/2`), peak `v*L.gain`.
  - `src.start(t + og, sp, grainSize * rate)`.
- Summed grains reconstruct the slice at the new duration with preserved pitch. No AudioWorklet — pure scheduled `BufferSource`s, consistent with the existing engine.

---

## 6. Lane management

- New **+ Chop** button in the `.rack` (next to `+ Lane`). Adds a chop lane (`chop1`, `chop2`, …) to **every** pattern (mirrors `addLane`'s cross-pattern behavior), with `slices:[0]`, `warp:'gate'`, empty steps. Tracked by a `chopCounter`.
- `defaultLanes()` seeds **one empty chop lane** (`id:'chop1'`, `name:'Chop 1'`) after the sample lanes so the feature is visible on first load.
- `addPat` already copies lane structure with empty steps and `buffer:null`; chop lanes must also reset `slices:[0]` in copied patterns (a freshly added pattern has no audio loaded).

---

## 7. Export & persistence

### 7.1 WAV (OfflineAudioContext path)

- Add `chopO(offline, t, v, L, stepIdx, dest)` replicating §5 (gate + granular) with offline-created `BufferSource`/`Gain` nodes. AudioBuffers decoded in the live `ac` are reusable by the offline context.
- Add `case 'chop'` to the offline `trigO` switch, and pass `stepIdx` through the offline render loop (currently `trigO(L,tt,vel)` → needs the step index for slice lookup + gate window). Gate window uses the same pure helper.

### 7.2 MIDI

- Each chop lane maps `slice index → baseNote + idx` (e.g. base `60`), exporting chops as a melodic track. Slot it into the existing per-lane note assignment alongside `DRUM_NOTE` / `SMP_NOTES`.

### 7.3 Persistence (localStorage)

- `saveCurrentPattern`: for chop lanes, serialize `slices`, `warp`, `semis`, `fname`, and steps (including `slice`). **Do not** serialize `buffer` (identical to current sample handling).
- `loadPattern`: restore `slices`/`warp`/`semis`/`fname`; set `buffer:null`. `coerceStep` restores per-step `slice`. Reloading the same audio file re-aligns to the saved normalized markers.
- `smpCounter`-style recompute: add a `chopCounter` recomputed from loaded lane ids on load.

---

## 8. QA plan (manual)

Run via `python -m http.server 8765`, open the file, and verify:
1. Load a loop into a chop lane; waveform renders.
2. Grid auto-slice (8) places 8 evenly-spaced markers; Map to steps fills steps 1–8 with slices 0–7.
3. Transient auto-slice on a breakbeat places markers on hits; sensitivity changes slice count.
4. Manual: click adds a marker, drag moves it, double-click deletes it.
5. Right-click a chop step → reassign its slice; playback reflects it.
6. Gate mode: slices play at native pitch, cut at the next hit; a lone slice holds to the bar.
7. Stretch mode: a slice placed sparsely stretches to fill the gap, pitch preserved, no audible clicks.
8. Patterns/song mode switch chop lanes correctly; mute/lane-gain work.
9. WAV export includes chop audio in both warp modes; MIDI export emits per-slice notes.
10. Save → reload page → load preset → reload the same audio file → markers + step assignments restored.

---

## 9. Out of scope (YAGNI)

- BPM auto-detection / global loop-warp-to-project-tempo (per-slice gate window already locks to grid; whole-loop tempo detection is a separate future feature).
- Per-slice individual pitch/envelope/reverse (lane-level `semis` only for now).
- Spectral (FFT/phase-vocoder) time-stretch — granular is the agreed pragmatic route.
- Drag-and-drop slices onto pads / dedicated pad bank UI (rejected integration option).
- Stereo-aware slicing (mono-mix for detection; playback uses the full buffer).
