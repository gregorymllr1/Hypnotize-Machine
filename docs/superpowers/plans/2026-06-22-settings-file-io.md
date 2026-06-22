# Settings File Import/Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `.json` settings export/import to the machine, complete the saved schema (master/tape/sidechain/lane length), and refactor save/load into a `buildSettings`/`applySettings` seam that pieces 2 & 3 will target.

**Architecture:** All changes in the single file `Hypnotize Machine.html`. Two new pure-ish functions (`buildSettings(title)`, `applySettings(obj)`) become the single source of truth for serialization/restoration; `saveCurrentPattern`/`loadPattern` become thin wrappers. Two header buttons add file download/upload on top.

**Tech Stack:** Vanilla JS, Blob/File APIs, existing `downloadBlob` helper. No build system, no dependencies, no test framework.

## Global Constraints

- Single file: `Hypnotize Machine.html`. No modules, bundlers, or dependencies.
- localStorage key stays `hypnotize_v2` (it's a map of saves; the per-save `version` field bumps to 3). Do NOT rename the key — that would orphan existing saves.
- Settings files do NOT embed audio (parity with localStorage). Sample/chop lanes carry `fname` only; `buffer:null` on load.
- Back-compat: `version:1` and `version:2` saves must still load.
- Verification is manual in-browser via `python -m http.server 8765`, plus a headless round-trip check at the end.
- Match the existing terse code style.

---

## File map

- **Modify only:** `Hypnotize Machine.html`
  - `<script>` — `coerceStep` (preserve `_timingOffset`), new `buildSettings`/`applySettings`, rewrite `saveCurrentPattern`/`loadPattern` as wrappers, add export/import handlers.
  - header markup — `Export File` / `Import File` buttons + hidden file input.

---

## Task 1: Schema completion + buildSettings/applySettings refactor

**Files:**
- Modify: `Hypnotize Machine.html` (`coerceStep`, `saveCurrentPattern`, `loadPattern`)

**Interfaces:**
- Produces: `buildSettings(title) -> settingsObj` (version:3, includes meta/master/tape/sidechain and per-lane `length`); `applySettings(obj) -> void` (applies any v1/v2/v3 object to the machine). `coerceStep` now preserves `_timingOffset`.

- [ ] **Step 1: Preserve `_timingOffset` through `coerceStep`.** Replace `coerceStep` (the `// --- step model ---` section):

```js
function coerceStep(s){ return (typeof s === 'number') ? makeStep(s) : {roll:s.roll||0, vel:s.vel!=null?s.vel:1, prob:s.prob!=null?s.prob:1, glide:s.glide!=null?s.glide:1, note:s.note!=null?s.note:36, slice:s.slice!=null?s.slice:-1, decay:s.decay!=null?s.decay:null, pan:s.pan!=null?s.pan:null, filter:s.filter!=null?s.filter:null, _timingOffset:s._timingOffset!=null?s._timingOffset:0}; }
```

- [ ] **Step 2: Add `buildSettings` and `applySettings`.** Insert both functions immediately after the `function saveStoredPatterns(p){...}` line (start of the persistence section), before `updatePresetDropdown`:

```js
function buildSettings(title){
  return {
    meta:{ app:'Hypnotize Machine', schema:3, title:title||'untitled', created:new Date().toISOString() },
    version:3,
    bpm, swing,
    master: parseFloat(volEl.value),
    tape: tapeAmt,
    sidechain: scAmt,
    patterns: patterns.map(p=>({
      name:p.name,
      lanes:p.lanes.map(L=>({
        id:L.id, name:L.name, type:L.type,
        steps:L.steps.map(s=>({...s})),
        gain:L.gain, mute:L.mute,
        tune:L.tune, decay:L.decay,
        semis:L.semis||0, fname:L.fname||null,
        slices:L.slices?[...L.slices]:undefined, warp:L.warp||undefined,
        length:L.length||16
      }))
    })),
    songSeq:[...songSeq], songMode
  };
}

function applySettings(obj){
  if(!obj) return;
  bpm=obj.bpm||140; swing=obj.swing||0;
  bpmEl.value=bpm; swingEl.value=swing;
  document.getElementById('bpmval').textContent=bpm;
  document.getElementById('bpmlbl').textContent=bpm;
  document.getElementById('swinglbl').textContent=Math.round(swing*100)+'%';

  const mv = obj.master!=null ? obj.master : 0.9;
  volEl.value=mv; document.getElementById('vollbl').textContent=Math.round(mv*100);
  if(master) master.gain.setTargetAtTime(mv, ac.currentTime, 0.01);
  tapeAmt = obj.tape!=null ? obj.tape : 0;
  tapeEl.value=tapeAmt; document.getElementById('tapelbl').textContent=Math.round(tapeAmt*100)+'%';
  if(ac) updateTape(tapeAmt);
  scAmt = obj.sidechain!=null ? obj.sidechain : 0;
  scEl.value=scAmt; document.getElementById('sclbl').textContent=Math.round(scAmt*100)+'%';

  if((obj.version===2||obj.version===3)&&obj.patterns){
    patterns=obj.patterns.map(pat=>({
      name:pat.name,
      lanes:pat.lanes.map(L=>({
        ...L,
        steps:L.steps.map(coerceStep),
        buffer:null,
        slices: L.type==='chop' ? (L.slices&&L.slices.length?[...L.slices]:[0]) : L.slices,
        warp:  L.type==='chop' ? (L.warp||'gate') : L.warp,
        length: L.length||16
      }))
    }));
    songSeq=obj.songSeq||[0];
    songMode=obj.songMode||false;
  } else {
    const rawLanes=(obj.lanes||[]).map(L=>({...L,steps:(L.steps||[]).map(coerceStep),buffer:null,length:L.length||16}));
    patterns=[{name:'A', lanes:rawLanes.length?rawLanes:defaultLanes()}];
    songSeq=[0]; songMode=false;
  }
  curPat=0; lanes=patterns[0].lanes;

  smpCounter=patterns.flatMap(p=>p.lanes).filter(l=>l.type==='sample')
    .reduce((mx,l)=>{ const n=parseInt(l.id.replace('smp',''))||0; return Math.max(mx,n); },2);
  chopCounter=patterns.flatMap(p=>p.lanes).filter(l=>l.type==='chop')
    .reduce((mx,l)=>{ const n=parseInt((l.id||'').replace('chop',''))||0; return Math.max(mx,n); },1);
  bassCounter=patterns.flatMap(p=>p.lanes).filter(l=>l.type==='bass808')
    .reduce((mx,l)=>{ const n=parseInt((l.id||'').replace('bass',''))||0; return Math.max(mx,n); },1);

  const songBtn=document.getElementById('songToggle');
  songBtn.textContent=songMode?'On':'Off'; songBtn.classList.toggle('on',songMode);
  document.getElementById('songSeqWrap').classList.toggle('show',songMode);

  rebuildGrid(); buildPatBtns(); buildSongSlots(); updatePresetDropdown();
}
```

- [ ] **Step 3: Rewrite `saveCurrentPattern` as a thin wrapper.** Replace the whole existing `saveCurrentPattern` function with:

```js
function saveCurrentPattern(){
  const name=(prompt('Save name:')||'').trim();
  if(!name) return;
  const saves=getStoredPatterns();
  saves[name]=buildSettings(name);
  saveStoredPatterns(saves);
  updatePresetDropdown();
  alert('Saved "'+name+'"');
}
```

- [ ] **Step 4: Rewrite `loadPattern` as a thin wrapper.** Replace the whole existing `loadPattern` function (from `function loadPattern(name){` through its closing `}`, including the bpm/version/counter/songBtn blocks that now live in `applySettings`) with:

```js
function loadPattern(name){
  const saves=getStoredPatterns();
  const p=saves[name]; if(!p) return;
  applySettings(p);
}
```

- [ ] **Step 5: Verify in browser.** Start `python -m http.server 8765`, open the page. Build a beat that exercises the schema: change BPM to 150, Swing up, Master/Tape/Sidechain knobs up, set a lane's **Len** to 12, right-click a step and set Pan/Filter, Humanize a lane. Click **Save**, name it "rt1". Reload the page. Pick "rt1" from **Load…**. Expected: BPM 150, knob positions, the Len=12 lane, pan/filter on that step, and humanize all restored. No console errors. (This proves the wider schema + refactor before any file I/O exists.)

- [ ] **Step 6: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(settings): complete schema + buildSettings/applySettings seam"
```

---

## Task 2: Export settings to file

**Files:**
- Modify: `Hypnotize Machine.html` (header markup + export handler)

**Interfaces:**
- Consumes: `buildSettings(title)`, `downloadBlob(blob, filename)`.
- Produces: `#exportSettings` button that downloads a `.hypnotize.json` file.

- [ ] **Step 1: Add the Export button.** In the header `.transport`, after the `delete` button (`<button id="delete" ...>×</button>`), add:

```html
        <button id="exportSettings" class="btn" title="Export settings to a .json file">Export File</button>
```

- [ ] **Step 2: Add the export handler.** Immediately after the existing `document.getElementById('delete').onclick=deletePattern;` line, add:

```js
document.getElementById('exportSettings').onclick=()=>{
  const currentTitle = document.getElementById('loadPreset').value || 'untitled';
  const title = ((prompt('Settings file name:', currentTitle) || '').trim()) || 'untitled';
  const obj = buildSettings(title);
  const fname = (title.toLowerCase().replace(/[^a-z0-9-_]+/g,'-').replace(/^-+|-+$/g,'') || 'untitled') + '.hypnotize.json';
  downloadBlob(new Blob([JSON.stringify(obj,null,2)],{type:'application/json'}), fname);
};
```

- [ ] **Step 3: Verify in browser.** Reload. Build/change a few settings. Click **Export File**, accept the prompt name. Expected: a `<name>.hypnotize.json` downloads. Open it in a text editor and confirm it contains `"meta"`, `"version": 3`, `"master"`, `"tape"`, `"sidechain"`, and each lane has `"length"`. No console errors.

- [ ] **Step 4: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(settings): export settings to .json file"
```

---

## Task 3: Import settings from file

**Files:**
- Modify: `Hypnotize Machine.html` (header markup + import handler)

**Interfaces:**
- Consumes: `applySettings(obj)`, `buildSettings(name)`, `getStoredPatterns()`, `saveStoredPatterns()`, `updatePresetDropdown()`.
- Produces: `#importSettings` button + `#importFile` input that loads a settings file and stores it into the localStorage library.

- [ ] **Step 1: Add the Import button + hidden file input.** In the header `.transport`, immediately after the `#exportSettings` button, add:

```html
        <button id="importSettings" class="btn" title="Import settings from a .json file">Import File</button>
        <input id="importFile" type="file" accept=".json,application/json" style="display:none">
```

- [ ] **Step 2: Add the import handler.** Immediately after the `#exportSettings` handler added in Task 2, add:

```js
document.getElementById('importSettings').onclick=()=>document.getElementById('importFile').click();
document.getElementById('importFile').onchange=(e)=>{
  const f=e.target.files[0]; if(!f){ return; }
  f.text().then(txt=>{
    let obj;
    try{ obj=JSON.parse(txt); }catch(_){ alert('Not a valid Hypnotize settings file'); return; }
    if(!obj || typeof obj!=='object' || (!obj.patterns && !obj.lanes)){ alert('Not a valid Hypnotize settings file'); return; }
    applySettings(obj);
    const name=((obj.meta&&obj.meta.title) || f.name.replace(/\.hypnotize\.json$/i,'').replace(/\.json$/i,'') || 'imported').trim() || 'imported';
    try{
      const saves=getStoredPatterns();
      saves[name]=buildSettings(name);
      saveStoredPatterns(saves);
      updatePresetDropdown();
      document.getElementById('loadPreset').value=name;
    }catch(err){ alert('Imported, but could not save to library: '+err.message); }
  });
  e.target.value='';
};
```

- [ ] **Step 3: Verify in browser.** Reload the page (fresh defaults). Click **Import File**, pick the `.hypnotize.json` exported in Task 2. Expected: the machine state matches the exported beat (BPM, knobs, lane length, p-locks, song sequence); the imported name now appears selected in the **Load…** dropdown and re-loads from there. Then import a deliberately-broken file (e.g. a `.json` containing `{}` or random text renamed to `.json`) → an `alert('Not a valid Hypnotize settings file')` fires with no state change. No console errors.

- [ ] **Step 4: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(settings): import settings from .json file + add to library"
```

---

## Self-review (against the spec)

**Spec coverage:**
- §3 schema completion (meta/master/tape/sidechain/length, version:3) → Task 1 `buildSettings`. ✓
- §3 back-compat v1/v2 → Task 1 `applySettings` version branches. ✓
- §4 `buildSettings`/`applySettings` seam + thin wrappers → Task 1. ✓
- §5 Export UI + download → Task 2. ✓
- §5 Import UI + validation + localStorage store + dropdown → Task 3. ✓
- §6 error handling (bad JSON, non-settings, partial fields) → Task 3 try/catch + validation; `applySettings` defaults. ✓
- §7 manual QA + round-trip → each task's verify step; final headless round-trip below. ✓
- §8 out of scope (audio embedding) honored — no audio bytes serialized. ✓
- Bonus gap fixed: `_timingOffset` now round-trips (Task 1 Step 1), making the spec's claim true.

**Placeholder scan:** No TBD/TODO; all steps carry full code. ✓

**Type/name consistency:** `buildSettings(title)`, `applySettings(obj)`, ids `#exportSettings`/`#importSettings`/`#importFile`, localStorage key `hypnotize_v2`, version `3` — consistent across tasks. ✓

## Final headless round-trip check (after Task 3)

Verify in the headless browser that `applySettings(buildSettings('t'))` preserves observable state and export→parse→apply is lossless for `bpm`, `master`, `tape`, `sidechain`, lane `length`s, and step `pan`/`filter`/`_timingOffset`. (This is my verification, not a committed test harness — manual QA per project convention.)
