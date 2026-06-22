# Audio Analysis Engine — Design (Piece 2 of 3)

**Date:** 2026-06-22
**Component:** `Hypnotize Machine.html` (single-file vanilla-JS Web Audio drum machine)
**Goal:** Analyze an uploaded audio file entirely in-browser and produce a structured **analysis report** describing its tempo, frequency-band energy build-up, onsets, layer entrances, and section structure. This is **piece 2 of 3** in the song-ingestion feature. It produces a report and a visualization; it does **not** modify machine state (that is piece 3).

---

## 1. Decisions (locked during brainstorming)

| # | Decision | Choice |
|---|----------|--------|
| 1 | Segmentation | **Self-similarity matrix (SSM) + checkerboard novelty** boundaries, with A/B repeat labels + heuristic type guesses. |
| 2 | Threading | **Main thread** with a staged progress label (FFT work is sub-second for typical songs). |
| 3 | Report display | **Canvas chart** (stacked band envelopes + onset curve + entrance/section/tempo overlays) **+ text readout**, plus Export Report JSON. |
| 4 | Envelope storage | Report stores **downsampled** envelope arrays (~800 pts). Exact tempo/entrance/onset/section values are full-precision regardless. |
| 5 | FFT | **Hand-rolled radix-2 FFT** (no dependency); analysis is offline over the decoded buffer, not via AnalyserNode. |
| 6 | Decode target | **Mono, 22050 Hz**, via `OfflineAudioContext` resample (Nyquist 11 kHz covers all six bands). |

---

## 2. Architecture & boundaries

A library of **pure DSP functions** (input → output, individually testable), one async **orchestrator** `analyzeAudio(file, onProgress) → report`, and a **UI panel** that renders the report. No machine-state changes — the report (a plain object, also exportable as JSON and kept in a `lastAnalysis` global) is the only output.

```
file → decodeToMono22k → computeSTFT ─┬→ bandEnvelopes → detectEntrances
                                       ├→ spectralFlux → pickOnsets → estimateTempo
                                       └→ coarseFeatures → noveltyCurve → pickBoundaries → buildSections
                                                                 ↓
                                              report → drawAnalysis + renderReadout + exportReport
```

All code lives in the single HTML file, in a clearly-delimited `// === AUDIO ANALYSIS ENGINE ===` section, following the existing pattern (like the chop DSP block).

---

## 3. Decode & STFT

### `decodeToMono22k(file) → Promise<{data:Float32Array, sampleRate:22050, duration}>`
`file.arrayBuffer()` → `ac.decodeAudioData` → render the decoded buffer through an `OfflineAudioContext(1, ceil(duration*22050), 22050)` (a single `BufferSource` → destination) to downmix to mono and resample to 22050. Returns channel 0 of the rendered buffer. Decode failure (unsupported/corrupt) rejects with a readable error.

### `fft(re, im) → void`
In-place radix-2 Cooley–Tukey on power-of-two arrays (`re`, `im` Float32/Float64). ~40 lines, no deps. Bit-reversal permutation + butterflies.

### `computeSTFT(data, sr, frameSize=1024, hop=512) → {frames, frameRate, freqPerBin, nBins}`
- Hann window per frame; zero-pad final partial frame.
- Per frame: copy windowed samples into `re` (im=0), `fft`, magnitude `sqrt(re²+im²)` for bins `0..frameSize/2` → `frames[i]` (Float32Array of `nBins = frameSize/2+1`).
- `frameRate = sr/hop` (~43.07 Hz), `freqPerBin = sr/frameSize` (~21.53 Hz).

---

## 4. Band energies & entrances

### Bands (Hz)
| name | lo | hi | instrument group (per doc) |
|------|----|----|----------------------------|
| sub | 30 | 90 | kick / 808 |
| low | 90 | 200 | kick body / 808 |
| lowmid | 200 | 600 | bass / snare body |
| mid | 600 | 2000 | vocals / melody |
| highmid | 2000 | 6000 | snare snap / perc / hats |
| high | 6000 | 11000 | hats / cymbals |

### `bandEnvelopes(frames, freqPerBin) → [{name, loHz, hiHz, env:Float32Array}]`
For each band, per frame, sum magnitudes of bins whose center freq falls in `[lo,hi)`. Then **smooth** each env with a moving average (~7 frames ≈ 160 ms) and **normalize** to 0–1 by its own max (guard max=0). `env.length === nFrames`.

### `detectEntrances(bandEnvs, frameRate, thresh=0.18, sustainSec=0.3) → returns entranceSec per band`
For each band, find the first frame `i` where `env[i..i+sustainFrames]` are all ≥ `thresh` (`sustainFrames = round(sustainSec*frameRate)`). `entranceSec = i/frameRate`, or `null` if never sustained. This is the "staircase" of layers entering.

---

## 5. Onsets & tempo

### `spectralFlux(frames) → Float32Array`
`flux[i] = Σ_bins max(0, mag_i[b] - mag_{i-1}[b])`, then normalize to 0–1 by max. `flux[0]=0`.

### `pickOnsets(flux, frameRate, minGapSec=0.09) → number[]`
Adaptive threshold per frame: `thr[i] = mean(window) + 1.0*std(window)` over a centered window (~0.5 s). An onset is a local maximum with `flux[i] > thr[i]`, spaced ≥ `minGapSec` from the previous. Returns onset times in seconds.

### `estimateTempo(flux, frameRate) → {primary, halfTime, doubleTime, candidates:[{bpm, strength}]}`
1. Compute autocorrelation of `flux` for lags corresponding to **40–240 BPM** (`lag = round(60*frameRate/bpm)`).
2. Find the strongest lag → `baseBpm`.
3. Build the **harmonic family** `{0.5, 1, 2, 3} × baseBpm` plus the top few raw ACF peak BPMs (dedupe, round).
4. Score each candidate = `acfStrength(lag)` × `prior(bpm)`, where `prior` is a Gaussian centered ~120 BPM (σ ≈ 50) — a gentle pull toward the perceptual comfort range that resolves octave errors (the 53.8→108, 68/139 cases from the notes).
5. `primary` = highest-scoring candidate within **70–180 BPM** (fallback: highest overall). `halfTime = primary/2`, `doubleTime = primary*2`, both surfaced explicitly. `candidates` = all scored, sorted desc, each with normalized `strength` 0–1.

---

## 6. SSM-novelty segmentation

### `coarseFeatures(frames, freqPerBin, frameRate, targetHz=3) → {features:Float32Array[], featRate}`
Downsample STFT frames by averaging groups of `round(frameRate/targetHz)` (~14) frames. For each coarse frame compute a **timbral feature**: ~12 log-spaced band energies spanning 30–11000 Hz, `log(1+E)` compressed, then **L2-normalized**. `featRate ≈ targetHz`. (A 3-min track → ~540 feature vectors.)

### `noveltyCurve(features, kernelSize=16) → Float32Array`
Slide a **Gaussian-tapered checkerboard kernel** (size `2*K`, `K=kernelSize`) along the SSM diagonal, computing similarities on the fly (no full N×N stored): for center `i`, `novelty[i] = Σ_{a,b∈[-K,K)} kernel(a,b) · cos(features[i+a], features[i+b])`, where `kernel` is +1 in same-side quadrants, −1 in cross quadrants, Gaussian-windowed. Edges clamped. Normalize 0–1.

### `pickBoundaries(novelty, featRate, minGapSec=3) → number[]`
Peak-pick novelty: local maxima above `mean+0.5*std`, spaced ≥ `minGapSec`. Convert to seconds. Always prepend `0` and append `duration` so sections tile the whole track.

### `buildSections(bounds, bandEnvs, frameRate, features, featRate) → [{startSec, endSec, label, typeGuess, activeBands, meanEnergy}]`
For each `[bounds[i], bounds[i+1])`:
- `activeBands` = band names whose mean env over the section ≥ 0.2.
- `meanEnergy` = mean of all bands' env over the section (0–1).
- `meanFeature` = average feature vector over the section's coarse frames.
- **Label:** greedily assign letters A,B,C…; a section whose `meanFeature` has cosine ≥ 0.9 with an earlier section's reuses that section's letter (A/B repetition), else a new letter.
- **typeGuess** heuristic: first section + low energy + few active bands → `intro`; rising active-band count vs previous → `build`; max-energy / most-active → `main`; sharp energy drop after a `main` → `break`; else `section`.

---

## 7. Report structure

```js
{
  meta:{ filename, durationSec, sampleRate:22050, analyzedAt:'<ISO>' },
  tempo:{ primary, halfTime, doubleTime, candidates:[{bpm, strength}] },
  bands:[ { name, loHz, hiHz, envelope:[…~800 pts, 0–1…], entranceSec|null, peak } ],
  onsets:{ envelope:[…~800 pts, 0–1…], times:[…sec, exact…], frameRate },
  sections:[ { startSec, endSec, label, typeGuess, activeBands:[…names…], meanEnergy } ]
}
```

`downsampleEnv(env, n=800) → number[]` averages `env` into ≤ `n` buckets. Exact data (`tempo`, `entranceSec`, `onsets.times`, section bounds) is never downsampled. The report is kept in a `lastAnalysis` global (consumed by piece 3) and exportable as `<filename>.analysis.json`.

---

## 8. UI / visualization

- **Analyze Audio** button in the header transport (near Import/Export File) opens `#analyzePanel` — a full-width overlay styled like the chop editor (`.show` toggle), with a close button.
- Panel contents:
  - **Choose audio file** input + the analyzed filename.
  - **Progress label** updated per stage: `Decoding… → Transform… → Bands & onsets… → Tempo… → Structure… → Done`.
  - **`<canvas id="analyzeCanvas">`** (~full width, ~320 px tall): the six band envelopes drawn as stacked filled areas (sub at bottom → high at top, the "staircase"), the onset curve as a thin line beneath, **vertical dashed entrance markers** (band-colored) at each `entranceSec`, **section dividers** with `label·typeGuess` text, and a faint **beat grid** at `tempo.primary`.
  - **Readout**: tempo (primary big, with `½ = …` / `2× = …` and candidate chips); a per-band **entrance table** (band · time · active?); a **sections list** (label · `mm:ss–mm:ss` · type · active bands).
  - **Export Report JSON** button.
- `drawAnalysis(report, canvas)` and `renderReadout(report, el)` are separate functions; the panel re-renders on resize.

---

## 9. Performance & error handling

- Orchestrator `analyzeAudio(file, onProgress)` is `async`; between heavy stages it `await`s a `0 ms` timeout so the progress label repaints (the work is sub-second for 3–4 min songs, but staged feedback reads as intentional).
- `decodeToMono22k` rejection → panel shows an inline error (`Could not decode this file`), no crash.
- Files longer than **720 s** → a soft `confirm()` warning before proceeding (memory/time).
- Empty/silent decode (max energy 0) → report still builds; envelopes are flat, tempo `candidates` may be empty, entrances all `null`; the panel notes "no clear rhythmic content."

---

## 10. Testing / QA plan

**Manual QA:** analyze a few real tracks (including a Memphis/phonk build-up like the one in the notes); eyeball that the staircase, tempo (with half/double), entrance times, and section dividers look sane.

**Headless verification of the pure functions** (against synthetic signals — factored as pure functions specifically to assert them):
1. `fft` — a unit sinusoid at bin `k` produces a magnitude peak at bin `k`.
2. `estimateTempo` — a synthetic onset envelope with clicks at exactly 120 BPM yields `primary` ≈ 120 (±2), with `halfTime`/`doubleTime` = 60/240; a 75-BPM click train resolves to 75 (not 37.5).
3. `detectEntrances` — synthetic band envelopes that ramp in at staggered times return entrance times matching the ramps (±1 frame).
4. `bandEnvelopes` — a pure 50 Hz tone puts energy in `sub`/`low` and ~0 in `high`; a 8 kHz tone does the reverse.
5. `noveltyCurve` — a feature sequence that switches timbre at the midpoint produces a novelty peak at the midpoint.

---

## 11. Out of scope (deferred)

- Mapping the report to machine settings / generating patterns, arrangement, effects (piece 3).
- Extemporaneous-sample extraction (piece 3).
- Beat-synchronous feature alignment / downbeat detection (the fixed-hop coarse features are sufficient for section novelty here).
- Key/chord detection (not needed for the drum-machine target).
- Web Worker offloading (main thread is fast enough; revisit only if profiling shows a problem).
