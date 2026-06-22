# Chop / Slice Sampler Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an SP-1200-style chop/slice sampler as a new `'chop'` lane type — load a loop, slice it (manual + transient/grid auto-slice), map slices to steps, and lock to the grid via per-lane gate or granular-stretch playback.

**Architecture:** All changes live in the single file `Hypnotize Machine.html` (CSS in `<style>`, markup in the body, logic in the one `<script>`). The chop lane reuses the existing lane/step/pattern/song/export/persistence machinery. New pure helpers (`detectTransients`, `gridSlices`, `gateWindow`, `mapSlicesToSteps`, `grainStretch`) are added so the DSP logic is isolated and reasoned about independently.

**Tech Stack:** Vanilla JS, Web Audio API (`AudioBufferSourceNode`, `GainNode`, `OfflineAudioContext`), `<canvas>` 2D. No build system, no dependencies, no test framework.

## Global Constraints

- Single file: `Hypnotize Machine.html`. Do not introduce modules, bundlers, or dependencies.
- `STEPS = 16` (existing global).
- Buffers are **never** persisted to localStorage (matches existing sample behavior) — only markers/warp/mapping persist; audio reloads from file.
- Granular time-stretch must use scheduled `AudioBufferSourceNode`s only — **no AudioWorklet** (consistent with the existing engine).
- Verification is **manual in-browser** via `python -m http.server 8765` (no automated tests). Each task ends with a concrete browser check.
- Match the existing terse code style (compact, semicolon-dense, `const`/`let`, no framework).

---

## File map

- **Modify only:** `Hypnotize Machine.html`
  - `<style>` block — chop editor + chop-cell CSS.
  - body markup — `+ Chop` button (rack), `#chopEditor` panel (after `.scroll`), slice row in `#stepCtrl` popover.
  - `<script>` — data model (`makeStep`/`coerceStep`/`defaultLanes`), chop state, `loadChopLoop`, chop editor (render + interaction + auto-slice), grid integration (`refreshCellVis`/`laneRow`/`showStepCtrl`), playback (`gateWindow`/`grainStretch`/`chopVoice`/`trigger`/`scheduleStep`), export (`chopO`/`trigO`/render loop/MIDI), persistence (`save`/`load`), lane management (`+ Chop`/`addPat`).

---

## Task 1: Data model + default chop lane

**Files:**
- Modify: `Hypnotize Machine.html` (step factories, `defaultLanes`, chop counter state)

**Interfaces:**
- Produces: step objects now carry `slice` (int, default `-1`). A chop lane object: `{id, name, type:'chop', gain, mute, buffer, fname, slices:[0], warp:'gate', semis:0, steps}`. Global `let chopCounter` (number, highest existing chopN).

- [ ] **Step 1: Add `slice` to the step factory.** Replace `makeStep` (currently line ~314):

```js
function makeStep(roll=0){ return {roll, vel:1.0, prob:1.0, glide:1, note:36, slice:-1}; }
```

- [ ] **Step 2: Carry `slice` through `coerceStep`** (line ~316):

```js
function coerceStep(s){ return (typeof s === 'number') ? makeStep(s) : {roll:s.roll||0, vel:s.vel!=null?s.vel:1, prob:s.prob!=null?s.prob:1, glide:s.glide!=null?s.glide:1, note:s.note!=null?s.note:36, slice:s.slice!=null?s.slice:-1}; }
```

- [ ] **Step 3: Seed one empty chop lane in `defaultLanes()`.** Add this object to the returned array, immediately after the `smp2` lane (before the closing `];`):

```js
    {id:'chop1', name:'Chop 1', type:'chop', gain:0.85, mute:false, buffer:null, fname:null,
     slices:[0], warp:'gate', semis:0,
     steps:stepsFrom(new Array(STEPS).fill(0))},
```

- [ ] **Step 4: Add the chop counter next to `smpCounter`** (line ~349, after `let smpCounter = 2;`):

```js
let chopCounter = 1;
```

- [ ] **Step 5: Verify in browser.** Start server (`python -m http.server 8765` in the project dir), open `http://localhost:8765/Hypnotize%20Machine.html`. Expected: page loads with **no console errors**, and a new lane row labeled **"Chop 1"** appears below "Sample 2" (16 step cells, currently bare). Existing lanes/playback unaffected.

- [ ] **Step 6: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(chop): add chop lane data model and default lane"
```

---

## Task 2: Chop lane row controls + lane management

**Files:**
- Modify: `Hypnotize Machine.html` (`laneRow` extra branch, `loadChopLoop`, `+ Chop` button + handler, `addPat` slice reset, `chopCounter` recompute on load)

**Interfaces:**
- Consumes: chop lane shape (Task 1).
- Produces: `loadChopLoop(L, file, fnameEl)` decodes audio into `L.buffer`; `toggleChopEditor(L)` (defined in Task 3 — referenced here by the Edit button; see note). New rack button `#addChop`.

> Note: the "Edit slices" button calls `toggleChopEditor(L)`, which is created in Task 3. After Task 2 the button exists but clicking it errors until Task 3 lands. That's the intended task seam — verify only the controls that work in this task (load, warp toggle, pitch, +Chop).

- [ ] **Step 1: Add the chop branch to `laneRow`'s `extra` chain.** In `laneRow`, the `if(L.type==='kick'){…} else if(L.type==='sample'){…}` block (lines ~741-752) gains a third branch. Add **before** `row.append(label,stepsEl,extra);`:

```js
  } else if(L.type==='chop'){
    const load=document.createElement('button'); load.className='load btn'; load.textContent='Load loop';
    const file=document.createElement('input'); file.type='file'; file.accept='audio/*'; file.style.display='none';
    const fname=document.createElement('span'); fname.className='fname'; fname.textContent=L.fname||'no loop';
    load.onclick=()=>file.click();
    file.onchange=()=>{ if(file.files[0]) loadChopLoop(L,file.files[0],fname); };
    const editBtn=document.createElement('button'); editBtn.className='btn'; editBtn.textContent='Edit slices ▾';
    editBtn.onclick=()=>toggleChopEditor(L);
    const warpBtn=document.createElement('button'); warpBtn.className='btn warp-btn';
    const setWarpLbl=()=>{ warpBtn.textContent='Warp: '+(L.warp==='stretch'?'Stretch':'Gate'); warpBtn.classList.toggle('on',L.warp==='stretch'); };
    setWarpLbl();
    warpBtn.onclick=()=>{ L.warp=(L.warp==='stretch'?'gate':'stretch'); setWarpLbl(); };
    extra.append(load,file,fname,editBtn,warpBtn);
    extra.append(miniRange('Pitch',-12,12,1,L.semis||0,v=>{L.semis=v;return (v>0?'+':'')+v;},(L.semis||0)>0?'+'+(L.semis||0):String(L.semis||0)));
```

The existing block ends with `}` then `row.append(...)`. The result reads `} else if(L.type==='sample'){ … } else if(L.type==='chop'){ … }`.

- [ ] **Step 2: Add `loadChopLoop` next to `loadSample`** (after `loadSample`, line ~775):

```js
function loadChopLoop(L,file,fnameEl){
  initAudio();
  fnameEl.textContent='decoding…';
  file.arrayBuffer().then(buf=>{
    ac.decodeAudioData(buf.slice(0),
      decoded=>{ L.buffer=decoded; L.fname=file.name; fnameEl.textContent=file.name;
        if(!L.slices||!L.slices.length) L.slices=[0];
        wfCache.id=null;
        if(chopEditorLaneId===L.id) drawChopEditor();
      },
      ()=>{ fnameEl.textContent='could not decode'; });
  });
}
```

> `wfCache`, `chopEditorLaneId`, `drawChopEditor` are defined in Task 3. After Task 2, loading a loop into a lane whose editor isn't open works (sets `L.buffer`/`fname`); the `wfCache`/`drawChopEditor` references only execute when the editor is open, which can't happen until Task 3. To avoid a ReferenceError on the `wfCache.id=null` line, **Task 3 must be applied before testing loop loading with the editor.** For this task's verification, confirm the filename label updates (decode success path sets `fnameEl.textContent=file.name` before touching `wfCache`).

Correction to avoid any TDZ risk in this task: guard the editor-only lines. Use this body instead:

```js
function loadChopLoop(L,file,fnameEl){
  initAudio();
  fnameEl.textContent='decoding…';
  file.arrayBuffer().then(buf=>{
    ac.decodeAudioData(buf.slice(0),
      decoded=>{ L.buffer=decoded; L.fname=file.name; fnameEl.textContent=file.name;
        if(!L.slices||!L.slices.length) L.slices=[0];
        if(typeof wfCache!=='undefined') wfCache.id=null;
        if(typeof chopEditorLaneId!=='undefined' && chopEditorLaneId===L.id) drawChopEditor();
      },
      ()=>{ fnameEl.textContent='could not decode'; });
  });
}
```

- [ ] **Step 3: Add the `+ Chop` rack button.** In the markup, after the `+ Lane` button (line ~251), add:

```html
      <button id="addChop" class="btn">+ Chop</button>
```

- [ ] **Step 4: Add the `+ Chop` handler.** After the `addLane` handler block (line ~879):

```js
document.getElementById('addChop').onclick=()=>{
  chopCounter++;
  const id='chop'+chopCounter;
  const name='Chop '+chopCounter;
  patterns.forEach(p=>{
    if(!p.lanes.find(l=>l.id===id)){
      p.lanes.push({id,name,type:'chop',gain:0.85,mute:false,buffer:null,fname:null,
        slices:[0],warp:'gate',semis:0,
        steps:stepsFrom(new Array(STEPS).fill(0))});
    }
  });
  rebuildGrid();
};
```

- [ ] **Step 5: Reset slices when `addPat` clones lanes.** In the `addPat` handler, change the `newLanes` map (lines ~846-850) to:

```js
  const newLanes=lanes.map(L=>({
    ...L,
    steps:stepsFrom(new Array(STEPS).fill(0)),
    buffer:null, fname:null,
    slices: L.type==='chop' ? [0] : L.slices
  }));
```

- [ ] **Step 6: Recompute `chopCounter` on load.** In `loadPattern`, after the `smpCounter` recompute block (line ~1178), add:

```js
  chopCounter=patterns.flatMap(p=>p.lanes)
    .filter(l=>l.type==='chop')
    .reduce((mx,l)=>{ const n=parseInt((l.id||'').replace('chop',''))||0; return Math.max(mx,n); },1);
```

- [ ] **Step 7: Verify in browser.** Reload. Expected: the **Chop 1** row now shows `Load loop`, `no loop`, `Edit slices ▾`, `Warp: Gate`, and a `Pitch` slider. Clicking **Warp** toggles `Gate`↔`Stretch` (button turns green on Stretch). Clicking **Load loop** and picking an audio file updates the filename label. Clicking **+ Chop** adds a "Chop 2" row. (Clicking "Edit slices" will error until Task 3 — expected.)

- [ ] **Step 8: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(chop): lane row controls, loop loading, +Chop button"
```

---

## Task 3: Slice editor panel + waveform rendering

**Files:**
- Modify: `Hypnotize Machine.html` (CSS, `#chopEditor` markup, chop editor state + render functions)

**Interfaces:**
- Consumes: chop lane shape, `lanes` global, `loadChopLoop` (Task 2).
- Produces: globals `chopEditorLaneId` (string|null), `wfCache` ({id,w,peaks}); functions `curChopLane()`, `toggleChopEditor(L)`, `closeChopEditor()`, `drawChopEditor()`, `getPeaks(L,w)`. The editor panel `#chopEditor` and `#chopCanvas`.

- [ ] **Step 1: Add chop editor CSS.** In the `<style>` block, after the `.icon-btn` rule (line ~192), add:

```css
  /* chop / slice editor */
  .chop-editor{display:none;flex-direction:column;gap:8px;padding:12px 18px;
    border-top:1px solid var(--line);background:rgba(0,0,0,.20);}
  .chop-editor.show{display:flex;}
  .chop-editor-head{display:flex;align-items:center;gap:12px;flex-wrap:wrap;}
  .chop-editor-title{font-size:11px;letter-spacing:.16em;text-transform:uppercase;color:var(--trim);}
  .chop-tools{display:flex;align-items:center;gap:8px;flex-wrap:wrap;margin-left:auto;}
  .chop-tools .mini span{min-width:auto;}
  .chop-canvas{width:100%;height:120px;background:var(--lcd-bg);border:1px solid #3a2a12;border-radius:8px;
    display:block;cursor:crosshair;touch-action:none;}
  .chop-hint{font-size:10px;color:var(--trim-dim);letter-spacing:.06em;}
  .warp-btn.on{background:#1a3520;border-color:#3a6040;color:#7ac97a;}
```

- [ ] **Step 2: Add the editor markup.** Immediately after the `.scroll` div that closes the grid (line ~257, after `</div>` of `.scroll`), insert:

```html
    <!-- chop / slice editor -->
    <div id="chopEditor" class="chop-editor">
      <div class="chop-editor-head">
        <span class="chop-editor-title" id="chopTitle">Slice Editor</span>
        <div class="chop-tools">
          <select id="chopMode" class="btn">
            <option value="transient">Transient</option>
            <option value="grid">Grid</option>
          </select>
          <span class="mini" id="chopSensWrap"><span>Sens</span><input type="range" id="chopSens" min="0" max="1" step="0.01" value="0.5"><b id="chopSensVal">50%</b></span>
          <select id="chopDiv" class="btn" style="display:none">
            <option value="4">4</option><option value="8" selected>8</option><option value="16">16</option><option value="32">32</option>
          </select>
          <button id="chopAuto" class="btn">Auto-slice</button>
          <button id="chopMap" class="btn">Map to steps</button>
          <button id="chopClear" class="btn">Clear slices</button>
          <button id="chopClose" class="btn icon-btn" title="Close">×</button>
        </div>
      </div>
      <canvas id="chopCanvas" class="chop-canvas"></canvas>
      <div class="chop-hint">Click = add marker · drag = move · double-click = delete</div>
    </div>
```

- [ ] **Step 3: Add editor state + render functions.** Add a new section in the `<script>` **before** the `// --- init ---` block (e.g. after the song-slots section, line ~865). The whole block:

```js
// --- chop / slice editor ---
let chopEditorLaneId=null;
let wfCache={id:null,w:0,peaks:null};
const chopEditor = document.getElementById('chopEditor');
const chopCanvas = document.getElementById('chopCanvas');
const chopCtx = chopCanvas.getContext('2d');

function curChopLane(){ return lanes.find(l=>l.id===chopEditorLaneId) || null; }

function toggleChopEditor(L){
  if(chopEditorLaneId===L.id && chopEditor.classList.contains('show')){ closeChopEditor(); return; }
  chopEditorLaneId=L.id; wfCache.id=null;
  document.getElementById('chopTitle').textContent='Slice Editor — '+L.name;
  chopEditor.classList.add('show');
  drawChopEditor();
}
function closeChopEditor(){ chopEditor.classList.remove('show'); chopEditorLaneId=null; }

function getPeaks(L,w){
  if(wfCache.id===L.id && wfCache.w===w && wfCache.peaks) return wfCache.peaks;
  const data=L.buffer.getChannelData(0);
  const step=Math.max(1,Math.floor(data.length/w));
  const peaks=new Float32Array(w*2);
  for(let x=0;x<w;x++){
    let min=1,max=-1; const start=x*step;
    for(let i=0;i<step;i++){ const v=data[start+i]||0; if(v<min)min=v; if(v>max)max=v; }
    peaks[x*2]=min; peaks[x*2+1]=max;
  }
  wfCache={id:L.id,w,peaks};
  return peaks;
}

function drawChopEditor(){
  const L=curChopLane();
  if(!L){ closeChopEditor(); return; }
  const w=Math.max(200, chopCanvas.clientWidth||chopCanvas.offsetWidth||600), h=120;
  if(chopCanvas.width!==w) chopCanvas.width=w;
  if(chopCanvas.height!==h) chopCanvas.height=h;
  chopCtx.clearRect(0,0,w,h);
  chopCtx.strokeStyle='rgba(243,177,60,.15)';
  chopCtx.beginPath(); chopCtx.moveTo(0,h/2); chopCtx.lineTo(w,h/2); chopCtx.stroke();
  if(L.buffer){
    const peaks=getPeaks(L,w);
    chopCtx.strokeStyle='rgba(243,177,60,.65)'; chopCtx.beginPath();
    for(let x=0;x<w;x++){
      const min=peaks[x*2], max=peaks[x*2+1];
      const y1=(1-(max*0.9+1)/2)*h, y2=(1-(min*0.9+1)/2)*h;
      chopCtx.moveTo(x+0.5,y1); chopCtx.lineTo(x+0.5,y2);
    }
    chopCtx.stroke();
  } else {
    chopCtx.fillStyle='rgba(216,203,178,.4)'; chopCtx.font='11px monospace';
    chopCtx.fillText('load a loop to slice', 12, h/2-8);
  }
  (L.slices||[]).forEach((pos,i)=>{
    const x=pos*w;
    chopCtx.strokeStyle = i===0 ? 'rgba(216,203,178,.5)' : 'rgba(239,127,44,.95)';
    chopCtx.beginPath(); chopCtx.moveTo(x,0); chopCtx.lineTo(x,h); chopCtx.stroke();
    chopCtx.fillStyle='#efb84a'; chopCtx.font='9px monospace';
    chopCtx.fillText(String(i), Math.min(w-10,x+3), 11);
  });
  if(playing && drawStep>=0){
    const st=L.steps[drawStep];
    if(st && st.roll>0 && st.slice>=0 && st.slice<L.slices.length){
      const x=L.slices[st.slice]*w;
      chopCtx.strokeStyle='rgba(122,201,122,.9)'; chopCtx.lineWidth=2;
      chopCtx.beginPath(); chopCtx.moveTo(x,0); chopCtx.lineTo(x,h); chopCtx.stroke();
      chopCtx.lineWidth=1;
    }
  }
}

window.addEventListener('resize',()=>{ if(chopEditor.classList.contains('show')){ wfCache.id=null; drawChopEditor(); } });
```

- [ ] **Step 4: Refresh editor inside `rebuildGrid`.** Append to `rebuildGrid` (line ~686-689) so pattern/song switches keep the editor consistent:

```js
function rebuildGrid(){
  gridEl.innerHTML = '';
  lanes.forEach(L => gridEl.append(laneRow(L)));
  if(chopEditor.classList.contains('show')){ if(curChopLane()){ wfCache.id=null; drawChopEditor(); } else closeChopEditor(); }
}
```

- [ ] **Step 5: Verify in browser.** Reload. Load an audio loop into Chop 1, click **Edit slices ▾**. Expected: the editor panel opens below the grid; the **waveform renders** on the dark canvas with a single start marker labeled `0`. Resizing the window redraws it crisply. Clicking **×** (close) hides it; clicking **Edit slices** on a lane with no loop shows "load a loop to slice".

- [ ] **Step 6: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(chop): slice editor panel with cached waveform render"
```

---

## Task 4: Slice markers — manual editing + auto-slice (transient/grid) + map

**Files:**
- Modify: `Hypnotize Machine.html` (pure slice functions, canvas pointer handlers, auto-slice control wiring)

**Interfaces:**
- Consumes: `curChopLane`, `drawChopEditor`, `chopCanvas`, chop lane shape.
- Produces: pure `detectTransients(data, sampleRate, sensitivity)→number[]`, `gridSlices(divisions)→number[]`, `mapSlicesToSteps(L)→void`. Wired controls `#chopMode/#chopSens/#chopDiv/#chopAuto/#chopMap/#chopClear/#chopClose`. Canvas add/move/delete-marker interaction.

- [ ] **Step 1: Add the pure slicing helpers.** In the chop editor section (after `closeChopEditor`), add:

```js
function gridSlices(divisions){ const a=[]; for(let i=0;i<divisions;i++) a.push(i/divisions); return a; }

function detectTransients(data, sampleRate, sensitivity){
  const hop=Math.max(1,Math.round(sampleRate*0.010));
  const n=Math.floor(data.length/hop);
  if(n<2) return [0];
  const rms=new Float32Array(n);
  for(let i=0;i<n;i++){ let sum=0; const s=i*hop; for(let j=0;j<hop;j++){ const v=data[s+j]||0; sum+=v*v; } rms[i]=Math.sqrt(sum/hop); }
  const flux=new Float32Array(n); let mx=0;
  for(let i=1;i<n;i++){ const d=rms[i]-rms[i-1]; flux[i]=d>0?d:0; if(flux[i]>mx)mx=flux[i]; }
  if(mx<=0) return [0];
  const thr=(1-sensitivity)*mx;
  const minGap=Math.max(1,Math.round(0.050*sampleRate/hop));
  const out=[0]; let last=-minGap;
  for(let i=1;i<n-1;i++){
    if(flux[i]>=thr && flux[i]>=flux[i-1] && flux[i]>=flux[i+1] && (i-last)>=minGap){
      out.push((i*hop)/data.length); last=i;
    }
  }
  const uniq=[...new Set(out.map(m=>+m.toFixed(5)))].sort((a,b)=>a-b).slice(0,64);
  if(uniq[0]!==0) uniq.unshift(0);
  return uniq;
}

function mapSlicesToSteps(L){
  const n=Math.min(STEPS,(L.slices||[]).length);
  for(let s=0;s<STEPS;s++){
    if(s<n){ L.steps[s].roll=1; L.steps[s].slice=s; }
    else { L.steps[s].roll=0; L.steps[s].slice=-1; }
  }
}
```

- [ ] **Step 2: Wire the auto-slice / map / clear controls.** Add after the helpers:

```js
const chopModeEl=document.getElementById('chopMode');
const chopSensEl=document.getElementById('chopSens');
const chopSensWrap=document.getElementById('chopSensWrap');
const chopDivEl=document.getElementById('chopDiv');
chopModeEl.onchange=()=>{
  const grid=chopModeEl.value==='grid';
  chopDivEl.style.display=grid?'':'none';
  chopSensWrap.style.display=grid?'none':'';
};
chopSensEl.oninput=()=>{ document.getElementById('chopSensVal').textContent=Math.round(chopSensEl.value*100)+'%'; };
document.getElementById('chopAuto').onclick=()=>{
  const L=curChopLane(); if(!L||!L.buffer) return;
  if(chopModeEl.value==='grid') L.slices=gridSlices(parseInt(chopDivEl.value));
  else L.slices=detectTransients(L.buffer.getChannelData(0),L.buffer.sampleRate,parseFloat(chopSensEl.value));
  mapSlicesToSteps(L); drawChopEditor(); rebuildGrid();
};
document.getElementById('chopMap').onclick=()=>{ const L=curChopLane(); if(!L) return; mapSlicesToSteps(L); rebuildGrid(); };
document.getElementById('chopClear').onclick=()=>{ const L=curChopLane(); if(!L) return; L.slices=[0]; L.steps.forEach(st=>{st.slice=-1;}); drawChopEditor(); rebuildGrid(); };
document.getElementById('chopClose').onclick=closeChopEditor;
```

- [ ] **Step 3: Add canvas marker interaction.** Add after the control wiring:

```js
let chopDrag=-1;
function chopCanvasPos(e){ const r=chopCanvas.getBoundingClientRect(); return Math.max(0,Math.min(1,(e.clientX-r.left)/r.width)); }
function chopNearestMarker(L,pos,tolPx){
  const w=chopCanvas.clientWidth||chopCanvas.width; let best=-1,bd=1e9;
  (L.slices||[]).forEach((p,i)=>{ const d=Math.abs(p-pos)*w; if(d<bd){bd=d;best=i;} });
  return bd<=tolPx?best:-1;
}
chopCanvas.addEventListener('pointerdown',(e)=>{
  const L=curChopLane(); if(!L||!L.buffer) return;
  const pos=chopCanvasPos(e); const hit=chopNearestMarker(L,pos,6);
  if(hit>0){ chopDrag=hit; try{ chopCanvas.setPointerCapture(e.pointerId); }catch(_){} }
  else if(hit===0){ /* start marker fixed */ }
  else { L.slices.push(pos); L.slices.sort((a,b)=>a-b); drawChopEditor(); }
});
chopCanvas.addEventListener('pointermove',(e)=>{
  if(chopDrag<0) return; const L=curChopLane(); if(!L) return;
  const pos=chopCanvasPos(e);
  const lo=(chopDrag-1>=0?L.slices[chopDrag-1]:0), hi=(chopDrag+1<L.slices.length?L.slices[chopDrag+1]:1);
  L.slices[chopDrag]=Math.max(lo+0.001,Math.min(hi-0.001,pos)); drawChopEditor();
});
chopCanvas.addEventListener('pointerup',()=>{ chopDrag=-1; });
chopCanvas.addEventListener('dblclick',(e)=>{
  const L=curChopLane(); if(!L) return;
  const hit=chopNearestMarker(L,chopCanvasPos(e),6);
  if(hit>0){ L.slices.splice(hit,1);
    L.steps.forEach(st=>{ if(st.slice>=L.slices.length) st.slice=L.slices.length-1; });
    drawChopEditor(); rebuildGrid();
  }
});
```

- [ ] **Step 4: Verify in browser.** Reload, load a loop, Edit slices. Expected:
  - **Grid mode** → set divisions to 8 → **Auto-slice** places 8 evenly spaced markers (0–7).
  - **Transient mode** → on a drum loop, Auto-slice places markers on hits; raising **Sens** adds markers, lowering removes them.
  - **Click** empty canvas adds a marker; **drag** a marker moves it (clamped between neighbors); **double-click** a marker deletes it (marker 0 stays).
  - **Clear slices** resets to one marker.
  (Step cells will populate after Task 5 makes them display slice numbers; for now confirm no console errors and markers behave.)

- [ ] **Step 5: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(chop): transient/grid auto-slice, manual marker editing, map-to-steps"
```

---

## Task 5: Step grid integration (slice display + per-step slice picker)

**Files:**
- Modify: `Hypnotize Machine.html` (`refreshCellVis` signature, `laneRow` chop cells, `#stepCtrl` slice row, `showStepCtrl`)

**Interfaces:**
- Consumes: chop lane shape, `refreshCellVis`.
- Produces: `refreshCellVis(c, st, chop)` (3rd arg optional bool); `showStepCtrl(cell, st, chop, L)` (extended); `#ctrlSlice` slice picker in the popover.

- [ ] **Step 1: Add chop-cell CSS** so the slice number shows even at `roll==1` (the existing `.cell[data-roll="1"] .num{display:none}` would hide it). Add **after** the existing `.cell` rules in `<style>` (after line ~131):

```css
  .cell.chop-cell[data-roll] .num{display:inline;}
```

- [ ] **Step 2: Extend `refreshCellVis` to render slice numbers.** Replace the function (lines ~691-706):

```js
function refreshCellVis(c, st, chop){
  c.dataset.roll = st.roll;
  c.textContent = '';
  if(chop && st.roll>0 && st.slice>=0){
    const x=document.createElement('span'); x.className='num'; x.textContent=String(st.slice)+(st.roll>1?'×'+st.roll:''); c.append(x);
  } else if(st.roll > 1){
    const x=document.createElement('span'); x.className='num'; x.textContent='×'+st.roll; c.append(x);
  } else {
    const d=document.createElement('span'); d.className='dot'; c.append(d);
  }
  if(st.roll > 0 && st.prob < 1){
    c.style.opacity = String(0.45 + 0.55 * st.prob);
    const m=document.createElement('span'); m.className='prob-mark'; m.textContent='~'; c.append(m);
  } else {
    c.style.opacity = '';
  }
}
```

- [ ] **Step 3: Make `laneRow` mark chop cells and pass the chop flag.** In the step-cell loop inside `laneRow` (lines ~723-738), replace the cell creation + handlers with:

```js
  for(let s=0;s<STEPS;s++){
    const c=document.createElement('button');
    c.className='cell'+(s%4===0&&s>0?' beat':'')+(L.type==='chop'?' chop-cell':'');
    c.dataset.step=s;
    c.setAttribute('aria-label', L.name+' step '+(s+1));
    refreshCellVis(c, L.steps[s], L.type==='chop');

    c.onclick=()=>{
      const st=L.steps[s];
      const idx=(ROLL_CYCLE.indexOf(st.roll)+1)%ROLL_CYCLE.length;
      st.roll=ROLL_CYCLE[idx];
      if(L.type==='chop' && st.roll>0 && st.slice<0){
        const ns=(L.slices||[]).length; st.slice = ns>0 ? (s%ns) : 0;
      }
      refreshCellVis(c, st, L.type==='chop');
    };
    c.oncontextmenu=(e)=>{ e.preventDefault(); showStepCtrl(c, L.steps[s], L.type==='chop', L); };
    stepsEl.append(c);
  }
```

- [ ] **Step 4: Add the slice row to the popover markup.** Inside `#stepCtrl` (line ~292), add this as the first `.mini` row (before the Vel row):

```html
  <div class="mini" id="ctrlSliceRow" style="display:none">
    <span>Slice</span>
    <input type="range" id="ctrlSlice" min="0" max="0" step="1" value="0">
    <b id="ctrlSliceVal">0</b>
  </div>
```

- [ ] **Step 5: Extend the popover JS.** Add refs + `activeChop` near the existing popover refs (line ~778-783):

```js
const ctrlSliceRow=document.getElementById('ctrlSliceRow');
const ctrlSlice=document.getElementById('ctrlSlice');
const ctrlSliceV=document.getElementById('ctrlSliceVal');
let activeChop=false;
```

Replace `showStepCtrl` (lines ~785-794) with:

```js
function showStepCtrl(cell, st, chop, L){
  if(!st || st.roll===0) return;
  activeStep=st; activeCell=cell; activeChop=!!chop;
  if(chop && L){
    const n=Math.max(1,(L.slices||[]).length);
    ctrlSliceRow.style.display='';
    ctrlSlice.max=String(n-1);
    if(st.slice<0) st.slice=0;
    if(st.slice>n-1) st.slice=n-1;
    ctrlSlice.value=String(st.slice);
    ctrlSliceV.textContent=String(st.slice);
  } else {
    ctrlSliceRow.style.display='none';
  }
  ctrlVel.value=st.vel; ctrlVelV.textContent=Math.round(st.vel*100)+'%';
  ctrlProb.value=st.prob; ctrlProbV.textContent=Math.round(st.prob*100)+'%';
  const r=cell.getBoundingClientRect();
  stepCtrl.style.left = Math.min(r.left, window.innerWidth-200)+'px';
  stepCtrl.style.top  = (r.bottom+5)+'px';
  stepCtrl.classList.add('show');
}
```

Add the slice input handler next to `ctrlVel.oninput`/`ctrlProb.oninput` (line ~795):

```js
ctrlSlice.oninput=()=>{
  if(!activeStep||!activeChop) return;
  activeStep.slice=parseInt(ctrlSlice.value);
  ctrlSliceV.textContent=String(activeStep.slice);
  if(activeCell) refreshCellVis(activeCell,activeStep,true);
};
```

- [ ] **Step 6: Verify in browser.** Reload, load loop, grid auto-slice into 8, **Map to steps** (or it auto-mapped from Auto-slice). Expected: Chop 1's first 8 step cells show **slice numbers 0–7**. Left-click a chop cell cycles roll (number stays, adds `×2` etc.). **Right-click** a chop cell → popover shows a **Slice** slider (0..n-1) above Vel/Prob; dragging it changes the displayed number on the cell. Right-click on a drum lane shows no slice row.

- [ ] **Step 7: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(chop): step cells show slice numbers + per-step slice picker"
```

---

## Task 6: Playback engine (gate + granular stretch)

**Files:**
- Modify: `Hypnotize Machine.html` (`gateWindow`, `grainStretch`, `chopVoice`, `trigger`, `scheduleStep`, draw-loop playhead)

**Interfaces:**
- Consumes: chop lane shape, `ac`, `scBus`, `master`, `stepDur`, `STEPS`.
- Produces: pure `gateWindow(steps, stepIdx, sd)→seconds`; `grainStretch(ctx, buffer, srcStart, srcLen, t, outDur, rate, peak, dest)→void`; `chopVoice(t, v, L, stepIdx)→void`. `trigger` gains `case 'chop'`; `scheduleStep` passes the step index.

- [ ] **Step 1: Add `gateWindow` and `grainStretch`.** Add just above the `trigger` function (line ~572):

```js
function gateWindow(steps, stepIdx, sd){
  for(let k=1;k<=STEPS;k++){
    const s=steps[(stepIdx+k)%STEPS];
    if(s.roll>0 && s.slice>=0) return k*sd;
  }
  return STEPS*sd;
}

function grainStretch(ctx, buffer, srcStart, srcLen, t, outDur, rate, peak, dest){
  const grain=0.08, hop=grain/2;
  const nG=Math.max(1,Math.min(200,Math.ceil(outDur/hop)));
  for(let k=0;k<nG;k++){
    const og=k*hop; if(og>=outDur+hop) break;
    const sp=srcStart + (outDur>0?(og/outDur):0)*srcLen;
    const s=ctx.createBufferSource(); s.buffer=buffer; s.playbackRate.value=rate;
    const g=ctx.createGain(); const a=grain/2; const start=t+og;
    g.gain.setValueAtTime(0.0001,start);
    g.gain.linearRampToValueAtTime(peak,start+a);
    g.gain.linearRampToValueAtTime(0.0001,start+grain);
    s.connect(g); g.connect(dest);
    s.start(start, Math.min(sp, buffer.duration-0.001), grain*rate);
    s.stop(start+grain+0.02);
  }
}
```

- [ ] **Step 2: Add `chopVoice`.** Add just below `bass808` (after line ~570, before `trigger`):

```js
function chopVoice(t,v,L,stepIdx){
  if(!L.buffer) return;
  const st=L.steps[stepIdx]; const idx=st.slice;
  if(idx<0 || idx>=L.slices.length) return;
  const dur=L.buffer.duration;
  const sliceStart=L.slices[idx]*dur;
  const sliceEnd=(idx+1<L.slices.length?L.slices[idx+1]:1)*dur;
  const sliceLen=Math.max(0.005, sliceEnd-sliceStart);
  const rate=Math.pow(2,(L.semis||0)/12);
  const sd=stepDur();
  const win = st.roll>1 ? sd/st.roll : gateWindow(L.steps, stepIdx, sd);
  const dest = scBus || master;
  const peak = Math.max(v*L.gain, 0.0003);
  if(L.warp==='stretch'){
    grainStretch(ac, L.buffer, sliceStart, sliceLen, t, win, rate, peak, dest);
  } else {
    const s=ac.createBufferSource(); s.buffer=L.buffer; s.playbackRate.value=rate;
    const g=ac.createGain();
    const playLen=Math.min(sliceLen, win*rate);
    const endT=t+playLen/rate;
    g.gain.setValueAtTime(0.0003,t);
    g.gain.linearRampToValueAtTime(peak,t+0.002);
    g.gain.setValueAtTime(peak, Math.max(t+0.002, endT-0.005));
    g.gain.linearRampToValueAtTime(0.0003,endT);
    s.connect(g); g.connect(dest);
    s.start(t, sliceStart, playLen); s.stop(endT+0.02);
  }
}
```

- [ ] **Step 3: Add `case 'chop'` to `trigger`** (line ~572-582):

```js
    case 'bass808':bass808(t,v,L,stepIdx); break;
    case 'chop':  chopVoice(t,v,L,stepIdx); break;
```

- [ ] **Step 4: Pass the step index in `scheduleStep`.** Change the live trigger call (line ~601) from `trigger(L, tt, vel);` to:

```js
        trigger(L, tt, vel, step);
```

- [ ] **Step 5: Playhead repaint.** In `draw()` (line ~638-653), after `setPlayhead(drawStep);` add:

```js
  if(chopEditor.classList.contains('show') && playing) drawChopEditor();
```

- [ ] **Step 6: Verify in browser.** Reload, load a loop into Chop 1, grid-slice into 8, Map to steps, press Play. Expected:
  - **Gate mode** (`Warp: Gate`): each step plays its slice at native pitch, cleanly, cut at the next step; a slice on step 1 with step 5 as the next hit holds ~4 steps. Changing **Pitch** repitches.
  - **Stretch mode** (`Warp: Stretch`): place one slice sparsely (e.g. only step 1 active) → the slice **stretches** to fill the gap, pitch preserved, no clicks/zipper noise.
  - The green **playhead** line tracks the playing slice in the editor.
  - No audible glitches; existing drum lanes still play.

- [ ] **Step 7: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(chop): gate + granular-stretch playback engine"
```

---

## Task 7: Export (WAV + MIDI) and persistence

**Files:**
- Modify: `Hypnotize Machine.html` (offline `chopO` + `trigO` + render loop, MIDI loop, `saveCurrentPattern`, `loadPattern`)

**Interfaces:**
- Consumes: `gateWindow`, `grainStretch` (global, reused with the offline context), chop lane shape.
- Produces: offline `chopO(t,v,L,stepIdx)`; `trigO(L,t,v,stepIdx)`; chop serialization in save/load.

- [ ] **Step 1: Add `chopO` to the offline render closure.** Inside the `export` click handler, after `smpO` (line ~968), add:

```js
    function chopO(t,v,L,stepIdx){
      if(!L.buffer) return;
      const st=L.steps[stepIdx]; const idx=st.slice;
      if(idx<0 || idx>=L.slices.length) return;
      const dur=L.buffer.duration;
      const sliceStart=L.slices[idx]*dur;
      const sliceEnd=(idx+1<L.slices.length?L.slices[idx+1]:1)*dur;
      const sliceLen=Math.max(0.005, sliceEnd-sliceStart);
      const rate=Math.pow(2,(L.semis||0)/12);
      const sd=stepDur();
      const win = st.roll>1 ? sd/st.roll : gateWindow(L.steps, stepIdx, sd);
      const peak=Math.max(v*L.gain,0.0003);
      if(L.warp==='stretch'){
        grainStretch(offline, L.buffer, sliceStart, sliceLen, t, win, rate, peak, mOff);
      } else {
        const s=offline.createBufferSource(); s.buffer=L.buffer; s.playbackRate.value=rate;
        const g=offline.createGain();
        const playLen=Math.min(sliceLen, win*rate);
        const endT=t+playLen/rate;
        g.gain.setValueAtTime(0.0003,t);
        g.gain.linearRampToValueAtTime(peak,t+0.002);
        g.gain.setValueAtTime(peak, Math.max(t+0.002, endT-0.005));
        g.gain.linearRampToValueAtTime(0.0003,endT);
        s.connect(g); g.connect(mOff);
        s.start(t, sliceStart, playLen); s.stop(endT+0.02);
      }
    }
```

- [ ] **Step 2: Pass step index through `trigO` and add the chop case.** Replace `trigO` (lines ~969-975):

```js
    function trigO(L,t,v,stepIdx){
      switch(L.type){
        case 'kick': kickO(t,v,L); break; case 'snare': snareO(t,v,L); break;
        case 'clap': clapO(t,v,L); break; case 'chh':  hatO(t,v,L,false); break;
        case 'ohh':  hatO(t,v,L,true); break; case 'sample': smpO(t,v,L); break;
        case 'chop': chopO(t,v,L,stepIdx); break;
      }
    }
```

- [ ] **Step 3: Pass `step` in the offline render loop.** Change the call (line ~989) from `trigO(L,tt,vel);` to:

```js
            trigO(L,tt,vel,step);
```

- [ ] **Step 4: Map chop slices to MIDI notes.** Replace the MIDI per-lane loop body (lines ~1045-1062) with:

```js
  for(const L of lanes){
    if(L.mute) continue;
    const isChop=L.type==='chop';
    const baseNote=isChop?60:(L.type==='sample'?SMP_NOTES[smpIdx++%SMP_NOTES.length]:DRUM_NOTE[L.type]);
    if(baseNote==null) continue;
    for(let s=0;s<STEPS;s++){
      const st=L.steps[s];
      if(st.roll===0) continue;
      if(isChop && st.slice<0) continue;
      const note=isChop?Math.max(0,Math.min(127,baseNote+st.slice)):baseNote;
      for(let i=0;i<st.roll;i++){
        const baseTick=s*tickStep;
        const swTicks=(s%2===1)?Math.round(swing*tickStep):0;
        const tick=baseTick+Math.round(i*tickStep/st.roll)+swTicks;
        const vel=Math.max(1,Math.round((st.roll===1?st.vel:st.vel*Math.max(0.45,1-i*0.13))*127));
        const offTick=tick+Math.max(1,Math.round(tickStep/st.roll)-2);
        evts.push([tick,0x99,note,vel]);
        evts.push([offTick,0x89,note,0]);
      }
    }
  }
```

- [ ] **Step 5: Serialize chop fields in `saveCurrentPattern`.** In the lane map (lines ~1131-1137), add `slices`/`warp` to the serialized object:

```js
      lanes:p.lanes.map(L=>({
        id:L.id, name:L.name, type:L.type,
        steps:L.steps.map(s=>({...s})),
        gain:L.gain, mute:L.mute,
        tune:L.tune, decay:L.decay,
        semis:L.semis||0, fname:L.fname||null,
        slices:L.slices?[...L.slices]:undefined, warp:L.warp||undefined
      }))
```

- [ ] **Step 6: Restore chop fields in `loadPattern`.** In the v2 patterns map (lines ~1157-1164), normalize chop fields:

```js
    patterns=p.patterns.map(pat=>({
      name:pat.name,
      lanes:pat.lanes.map(L=>({
        ...L,
        steps:L.steps.map(coerceStep),
        buffer:null,
        slices: L.type==='chop' ? (L.slices&&L.slices.length?[...L.slices]:[0]) : L.slices,
        warp:  L.type==='chop' ? (L.warp||'gate') : L.warp
      }))
    }));
```

- [ ] **Step 7: Verify in browser.** Reload, build a chopped loop on Chop 1.
  - **Export WAV** in Gate mode → downloaded WAV contains the chopped audio in time. Switch to **Stretch**, export again → stretched chops render.
  - **Export MIDI** → load the `.mid` in any DAW/player: the chop lane emits notes starting at C3 (60) ascending per slice index.
  - **Save** a preset → reload the page → **Load** the preset → re-load the same audio file into Chop 1 → the markers and step→slice mapping are intact, warp mode restored.

- [ ] **Step 8: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(chop): WAV/MIDI export + persistence for chop lanes"
```

---

## Self-review (against the spec)

**Spec coverage:**
- §2 Data model → Task 1 (slice field, chop lane, counter). ✓
- §3.1 lane row controls → Task 2. ✓
- §3.2 step interaction (left-click default slice, right-click slice picker) → Task 5. ✓
- §3.3 editor panel + waveform + marker editing → Tasks 3, 4. ✓
- §4 transient detection → Task 4 (`detectTransients`) + grid (`gridSlices`). ✓
- §5.1 gate window (incl. roll subdivision) → Task 6 (`gateWindow`, `win` selection). ✓
- §5.2 gate + granular stretch playback → Task 6 (`chopVoice`, `grainStretch`). ✓
- §6 lane management (+Chop, default lane, addPat reset) → Tasks 1, 2. ✓
- §7.1 WAV offline → Task 7 (`chopO`, `trigO`, render loop). ✓
- §7.2 MIDI slice→note → Task 7. ✓
- §7.3 persistence (no buffer, restore markers, chopCounter) → Tasks 2, 7. ✓
- §8 manual QA → every task's verify step + final pass. ✓

**Placeholder scan:** No TBD/TODO; all steps carry full code. ✓

**Type/name consistency:** `slice` field, `slices` array, `warp` string, `chopEditorLaneId`, `wfCache`, `curChopLane`, `drawChopEditor`, `gateWindow(steps,stepIdx,sd)`, `grainStretch(ctx,buffer,srcStart,srcLen,t,outDur,rate,peak,dest)`, `chopVoice(t,v,L,stepIdx)`, `trigger(L,t,v,stepIdx)`, `trigO(L,t,v,stepIdx)`, `chopO(t,v,L,stepIdx)` — names match across tasks. ✓

**Known task seam:** Task 2 references `wfCache`/`chopEditorLaneId`/`drawChopEditor` (defined in Task 3); the references are guarded with `typeof` checks so loop loading works pre-Task-3, and full editor behavior is verified in Task 3+. ✓
