# Audio Analysis Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a full in-browser audio analysis engine to `Hypnotize Machine.html` that decodes an uploaded audio file to mono 22050 Hz, runs STFT-based DSP (band envelopes, onset detection, tempo, SSM-novelty segmentation), and renders the structured report as a canvas chart + text readout inside an `#analyzePanel` overlay.

**Architecture:** All code goes in the single HTML file in a clearly-delimited `// === AUDIO ANALYSIS ENGINE ===` block inserted before `// --- init ---`. The DSP functions are pure (no DOM/AudioContext dependency), making them console-testable with synthetic data. The UI overlay is styled identically to the chop editor (same `.show` toggle, same `--bg`/`--panel` variables). The report is stored in a `lastAnalysis` global consumed by Piece 3.

**Tech Stack:** Vanilla JS, Web Audio API (`OfflineAudioContext`), Canvas 2D API, no build system, no test runner.

## Global Constraints

- Single file: `Hypnotize Machine.html` (currently 1889 lines). No modules, bundlers, or dependencies.
- All new JS goes inside the existing `<script>` block, in a `// === AUDIO ANALYSIS ENGINE ===` section, inserted between line 1882 and `// --- init ---`.
- All new CSS goes inside the existing `<style>` block, after `.warp-btn.on` (around line 207).
- No machine-state mutation. The engine reads files and writes to `lastAnalysis`; it does NOT call `applySettings` or modify `patterns`/`lanes` (that is Piece 3).
- `downloadBlob(blob, filename)` already exists at line 1464 — reuse it.
- `lastAnalysis` is a module-level `let` set by `analyzeAudio`; Piece 3 will read it.
- Verification is manual browser QA + browser-console synthetic-data checks. No test harness.
- Match the existing terse code style (single-line functions, no comments on obvious code).

---

## File map

**Modify only:** `Hypnotize Machine.html`

| Region | What changes |
|--------|--------------|
| `<style>` block, after line 207 (`.warp-btn.on`) | Add `.analyze-panel` / `.analyze-canvas` / `.analyze-readout` CSS |
| Header `<div class="transport">`, after line 244 (`<input id="importFile"...>`) | Add `<button id="analyzeAudio">` |
| After chop editor `</div>` (line 300), before `<!-- song / pattern bar -->` (line 302) | Add `#analyzePanel` overlay HTML |
| `<script>`, after line 1882 (end of chop canvas events), before `// --- init ---` (line 1884) | Add entire `// === AUDIO ANALYSIS ENGINE ===` block |

---

## Task 1: Pure DSP core

**Files:**
- Modify: `Hypnotize Machine.html` (JS — add `// === AUDIO ANALYSIS ENGINE ===` section header + 7 pure functions)

**Interfaces:**
- Produces:
  - `fft(re:Float32Array, im:Float32Array) → void` (in-place)
  - `computeSTFT(data:Float32Array, sr:number, frameSize?:number, hop?:number) → {frames:Float32Array[], frameRate:number, freqPerBin:number, nBins:number}`
  - `BANDS:[{name,loHz,hiHz}]` (6-element const array)
  - `bandEnvelopes(frames:Float32Array[], freqPerBin:number) → [{name,loHz,hiHz,env:Float32Array}]`
  - `detectEntrances(bandEnvs:[{env}], frameRate:number, thresh?:number, sustainSec?:number) → (number|null)[]`
  - `spectralFlux(frames:Float32Array[]) → Float32Array`
  - `pickOnsets(flux:Float32Array, frameRate:number, minGapSec?:number) → number[]`
  - `estimateTempo(flux:Float32Array, frameRate:number) → {primary:number, halfTime:number, doubleTime:number, candidates:{bpm:number,strength:number}[]}`

- [ ] **Step 1: Insert the DSP section header and `fft`.** After line 1882 (after `chopCanvas.addEventListener('dblclick', ...` block closes), before `// --- init ---`, insert:

```js
// === AUDIO ANALYSIS ENGINE ===

function fft(re, im){
  const n=re.length;
  let j=0;
  for(let i=1;i<n;i++){
    let bit=n>>1; for(;j&bit;bit>>=1)j^=bit; j^=bit;
    if(i<j){ let t=re[i];re[i]=re[j];re[j]=t; t=im[i];im[i]=im[j];im[j]=t; }
  }
  for(let len=2;len<=n;len<<=1){
    const ang=-2*Math.PI/len, wr=Math.cos(ang), wi=Math.sin(ang);
    for(let i=0;i<n;i+=len){
      let cr=1,ci=0;
      for(let k=0;k<len/2;k++){
        const ur=re[i+k],ui=im[i+k];
        const vr=re[i+k+len/2]*cr-im[i+k+len/2]*ci;
        const vi=re[i+k+len/2]*ci+im[i+k+len/2]*cr;
        re[i+k]=ur+vr; im[i+k]=ui+vi;
        re[i+k+len/2]=ur-vr; im[i+k+len/2]=ui-vi;
        const ncr=cr*wr-ci*wi; ci=cr*wi+ci*wr; cr=ncr;
      }
    }
  }
}
```

- [ ] **Step 2: Add `computeSTFT`.** Immediately after `fft`:

```js
function computeSTFT(data, sr, frameSize=1024, hop=512){
  const nBins=frameSize/2+1, nFrames=Math.ceil(data.length/hop);
  const hann=new Float32Array(frameSize);
  for(let i=0;i<frameSize;i++) hann[i]=0.5*(1-Math.cos(2*Math.PI*i/(frameSize-1)));
  const re=new Float32Array(frameSize), im=new Float32Array(frameSize);
  const frames=[];
  for(let f=0;f<nFrames;f++){
    re.fill(0); im.fill(0);
    const start=f*hop;
    for(let i=0;i<frameSize;i++) re[i]=(data[start+i]||0)*hann[i];
    fft(re,im);
    const mag=new Float32Array(nBins);
    for(let b=0;b<nBins;b++) mag[b]=Math.sqrt(re[b]*re[b]+im[b]*im[b]);
    frames.push(mag);
  }
  return {frames, frameRate:sr/hop, freqPerBin:sr/frameSize, nBins};
}
```

- [ ] **Step 3: Add `BANDS`, `bandEnvelopes`, and `detectEntrances`.** Immediately after:

```js
const BANDS=[
  {name:'sub',loHz:30,hiHz:90},
  {name:'low',loHz:90,hiHz:200},
  {name:'lowmid',loHz:200,hiHz:600},
  {name:'mid',loHz:600,hiHz:2000},
  {name:'highmid',loHz:2000,hiHz:6000},
  {name:'high',loHz:6000,hiHz:11000}
];

function bandEnvelopes(frames, freqPerBin){
  const n=frames.length;
  return BANDS.map(({name,loHz,hiHz})=>{
    const lo=Math.round(loHz/freqPerBin), hi=Math.round(hiHz/freqPerBin);
    const env=new Float32Array(n);
    for(let f=0;f<n;f++){ let s=0; for(let b=lo;b<hi&&b<frames[f].length;b++) s+=frames[f][b]; env[f]=s; }
    const sm=3, out=new Float32Array(n);
    for(let f=0;f<n;f++){ let s=0,c=0; for(let k=f-sm;k<=f+sm;k++){if(k>=0&&k<n){s+=env[k];c++;}} out[f]=s/c; }
    let mx=0; for(let f=0;f<n;f++) if(out[f]>mx) mx=out[f];
    if(mx>0) for(let f=0;f<n;f++) out[f]/=mx;
    return {name,loHz,hiHz,env:out};
  });
}

function detectEntrances(bandEnvs, frameRate, thresh=0.18, sustainSec=0.3){
  const sf=Math.round(sustainSec*frameRate);
  return bandEnvs.map(({env})=>{
    const n=env.length;
    for(let i=0;i<n-sf;i++){
      let ok=true; for(let k=0;k<sf;k++) if(env[i+k]<thresh){ok=false;break;}
      if(ok) return i/frameRate;
    }
    return null;
  });
}
```

- [ ] **Step 4: Add `spectralFlux`, `pickOnsets`, `estimateTempo`.** Immediately after:

```js
function spectralFlux(frames){
  const n=frames.length, flux=new Float32Array(n); let mx=0;
  for(let f=1;f<n;f++){
    let s=0; for(let b=0;b<frames[f].length;b++){ const d=frames[f][b]-frames[f-1][b]; if(d>0) s+=d; }
    flux[f]=s; if(s>mx) mx=s;
  }
  if(mx>0) for(let f=0;f<n;f++) flux[f]/=mx;
  return flux;
}

function pickOnsets(flux, frameRate, minGapSec=0.09){
  const n=flux.length, wf=Math.round(0.5*frameRate), minGap=Math.round(minGapSec*frameRate);
  const onsets=[]; let last=-minGap;
  for(let i=1;i<n-1;i++){
    let sum=0,sum2=0,cnt=0;
    for(let k=i-wf;k<=i+wf;k++) if(k>=0&&k<n){sum+=flux[k];sum2+=flux[k]*flux[k];cnt++;}
    const mean=sum/cnt, std=Math.sqrt(Math.max(0,sum2/cnt-mean*mean));
    if(flux[i]>mean+std && flux[i]>flux[i-1] && flux[i]>=flux[i+1] && (i-last)>=minGap){
      onsets.push(i/frameRate); last=i;
    }
  }
  return onsets;
}

function estimateTempo(flux, frameRate){
  const lagMin=Math.round(60*frameRate/240), lagMax=Math.round(60*frameRate/40), n=flux.length;
  const acf=new Float32Array(lagMax+1);
  for(let lag=lagMin;lag<=lagMax&&lag<n;lag++){
    let s=0; for(let i=0;i+lag<n;i++) s+=flux[i]*flux[i+lag]; acf[lag]=s;
  }
  const peaks=[]; for(let l=lagMin+1;l<lagMax;l++) if(acf[l]>acf[l-1]&&acf[l]>=acf[l+1]) peaks.push(l);
  peaks.sort((a,b)=>acf[b]-acf[a]);
  const top=peaks.slice(0,5);
  if(!top.length) return {primary:120,halfTime:60,doubleTime:240,candidates:[]};
  const prior=bpm=>Math.exp(-(bpm-120)**2/(2*50**2));
  const cands=new Map();
  for(const lag of top){
    for(const m of [0.5,1,2,3]){
      const bpm=Math.round(60*frameRate/(lag/m));
      if(bpm<35||bpm>300) continue;
      const score=acf[lag]*prior(bpm);
      if(!cands.has(bpm)||cands.get(bpm)<score) cands.set(bpm,score);
    }
  }
  const sorted=[...cands.entries()].sort((a,b)=>b[1]-a[1]);
  const maxS=sorted[0]?sorted[0][1]:1;
  const candidates=sorted.map(([bpm,s])=>({bpm,strength:s/maxS}));
  const primary=(candidates.find(c=>c.bpm>=70&&c.bpm<=180)||candidates[0]||{bpm:120}).bpm;
  return {primary,halfTime:Math.round(primary/2),doubleTime:primary*2,candidates};
}
```

- [ ] **Step 5: Verify in browser console.** Open `http://localhost:8765`. In DevTools console:

```js
// FFT: unit sinusoid at bin 4 (n=16 keeps it short; real use is 1024)
const N=16, re=new Float32Array(N), im=new Float32Array(N);
for(let i=0;i<N;i++) re[i]=Math.cos(2*Math.PI*4*i/N);
fft(re,im);
const mag4=Math.sqrt(re[4]**2+im[4]**2);
console.assert(mag4>N*0.9, 'FFT peak at bin 4 expected, got '+mag4);

// Tempo: 120 BPM click train → primary ≈ 120
const sr=22050, hop=512, fr=sr/hop, dur=30;
const fl=new Float32Array(Math.ceil(dur*fr));
const beatLag=Math.round(60*fr/120);
for(let i=0;i<fl.length;i+=beatLag) fl[i]=1;
const t=estimateTempo(fl,fr);
console.assert(Math.abs(t.primary-120)<=2,'Expected 120, got '+t.primary);
console.assert(t.halfTime===60&&t.doubleTime===240,'Half/double wrong');

// Band separation: 50 Hz sine → energy in sub, near-zero in high
const n=22050, d=new Float32Array(n);
for(let i=0;i<n;i++) d[i]=Math.sin(2*Math.PI*50*i/22050);
const {frames,freqPerBin}=computeSTFT(d,22050);
const bands=bandEnvelopes(frames,freqPerBin);
const subPeak=Math.max(...bands[0].env);
const hiPeak=Math.max(...bands[5].env);
console.assert(subPeak>0.9,'sub band should be near 1, got '+subPeak);
console.assert(hiPeak<0.2,'high band should be ~0, got '+hiPeak);

// Entrance detection: ramp starting at 2s
const envTest=new Float32Array(200); envTest.fill(0,0,86); envTest.fill(1,86);
const entrance=detectEntrances([{env:envTest}],43.07)[0];
console.assert(entrance!==null&&Math.abs(entrance-2.0)<0.1,'Expected ~2s entrance, got '+entrance);

console.log('Task 1 assertions passed');
```

Expected: four assertions pass, `'Task 1 assertions passed'` logged, no exceptions.

- [ ] **Step 6: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(analysis): DSP core — fft, STFT, band envelopes, flux, onsets, tempo"
```

---

## Task 2: SSM-novelty segmentation

**Files:**
- Modify: `Hypnotize Machine.html` (JS — 4 more pure functions inside the `// === AUDIO ANALYSIS ENGINE ===` block)

**Interfaces:**
- Consumes: `bandEnvelopes` output, `computeSTFT` output (`frames`, `freqPerBin`, `frameRate`)
- Produces:
  - `coarseFeatures(frames:Float32Array[], freqPerBin:number, frameRate:number, targetHz?:number) → {features:Float32Array[], featRate:number}`
  - `noveltyCurve(features:Float32Array[], kernelSize?:number) → Float32Array`
  - `pickBoundaries(novelty:Float32Array, featRate:number, duration:number, minGapSec?:number) → number[]`
  - `buildSections(bounds:number[], bandEnvs:[{name,env}], frameRate:number, features:Float32Array[], featRate:number) → {startSec,endSec,label,typeGuess,activeBands:string[],meanEnergy:number}[]`

- [ ] **Step 1: Add `coarseFeatures`.** Immediately after `estimateTempo`:

```js
function coarseFeatures(frames, freqPerBin, frameRate, targetHz=3){
  const gs=Math.max(1,Math.round(frameRate/targetHz)), nC=Math.ceil(frames.length/gs);
  const nB=12, loHz=30, hiHz=11000;
  const edges=[];
  for(let i=0;i<=nB;i++) edges.push(loHz*Math.pow(hiHz/loHz,i/nB));
  const binEdges=edges.map(h=>Math.min(frames[0].length-1,Math.round(h/freqPerBin)));
  const features=[];
  for(let g=0;g<nC;g++){
    const s=g*gs, e=Math.min(frames.length,(g+1)*gs);
    const feat=new Float32Array(nB);
    for(let f=s;f<e;f++) for(let b=0;b<nB;b++){ let v=0; for(let k=binEdges[b];k<binEdges[b+1];k++) v+=frames[f][k]; feat[b]+=v; }
    for(let b=0;b<nB;b++) feat[b]=Math.log1p(feat[b]/(e-s));
    let norm=0; for(let b=0;b<nB;b++) norm+=feat[b]*feat[b]; norm=Math.sqrt(norm)||1;
    for(let b=0;b<nB;b++) feat[b]/=norm;
    features.push(feat);
  }
  return {features, featRate:frameRate/gs};
}
```

- [ ] **Step 2: Add `noveltyCurve`.** Immediately after:

```js
function noveltyCurve(features, kernelSize=16){
  const K=kernelSize, n=features.length, novelty=new Float32Array(n);
  const gw=new Float32Array(K);
  for(let i=0;i<K;i++) gw[i]=Math.exp(-0.5*(i/(K*0.5))**2);
  function cosSim(a,b){ let s=0; for(let i=0;i<a.length;i++) s+=a[i]*b[i]; return s; }
  for(let ci=K;ci<n-K;ci++){
    let nov=0;
    for(let da=-K;da<K;da++){
      for(let db=-K;db<K;db++){
        const ia=ci+da, ib=ci+db;
        if(ia<0||ia>=n||ib<0||ib>=n) continue;
        const w=gw[Math.abs(da)]*gw[Math.abs(db)];
        const sign=((da<0)===(db<0))?1:-1;
        nov+=sign*w*cosSim(features[ia],features[ib]);
      }
    }
    novelty[ci]=nov;
  }
  let mn=Infinity,mx=-Infinity;
  for(let i=K;i<n-K;i++){ if(novelty[i]<mn)mn=novelty[i]; if(novelty[i]>mx)mx=novelty[i]; }
  const range=(mx-mn)||1;
  for(let i=0;i<n;i++) novelty[i]=Math.max(0,(novelty[i]-mn)/range);
  return novelty;
}
```

- [ ] **Step 3: Add `pickBoundaries` and `buildSections`.** Immediately after:

```js
function pickBoundaries(novelty, featRate, duration, minGapSec=3){
  const n=novelty.length, minGap=Math.round(minGapSec*featRate);
  let sum=0,sum2=0;
  for(let i=0;i<n;i++){sum+=novelty[i];sum2+=novelty[i]**2;}
  const mean=sum/n, std=Math.sqrt(Math.max(0,sum2/n-mean**2));
  const thr=mean+0.5*std, bounds=[0]; let last=0;
  for(let i=1;i<n-1;i++){
    if(novelty[i]>thr&&novelty[i]>novelty[i-1]&&novelty[i]>=novelty[i+1]&&(i-last)>=minGap){
      bounds.push(i/featRate); last=i;
    }
  }
  bounds.push(duration);
  return bounds;
}

function buildSections(bounds, bandEnvs, frameRate, features, featRate){
  const LABELS='ABCDEFGHIJKLMNOPQRSTUVWXYZ', protos=[], sections=[]; let li=0;
  for(let i=0;i<bounds.length-1;i++){
    const ss=bounds[i], se=bounds[i+1];
    const sf=Math.round(ss*frameRate), ef=Math.round(se*frameRate);
    const sfc=Math.round(ss*featRate), efc=Math.round(se*featRate);
    const activeBands=bandEnvs.filter(({env})=>{
      let s=0,c=0; for(let f=sf;f<ef&&f<env.length;f++){s+=env[f];c++;} return c>0&&s/c>=0.2;
    }).map(b=>b.name);
    let te=0,tc=0;
    bandEnvs.forEach(({env})=>{ for(let f=sf;f<ef&&f<env.length;f++){te+=env[f];tc++;} });
    const meanEnergy=tc>0?te/tc:0;
    const mf=new Float32Array(features[0]?features[0].length:12); let fc=0;
    for(let f=sfc;f<efc&&f<features.length;f++){for(let b=0;b<mf.length;b++) mf[b]+=features[f][b];fc++;}
    if(fc>0) for(let b=0;b<mf.length;b++) mf[b]/=fc;
    let label=null;
    for(const p of protos){ let d=0; for(let b=0;b<mf.length;b++) d+=mf[b]*p.feat[b]; if(d>=0.9){label=p.label;break;} }
    if(!label){ label=LABELS[li%26]; li++; protos.push({label,feat:mf}); }
    sections.push({startSec:ss,endSec:se,label,typeGuess:'section',activeBands,meanEnergy});
  }
  const maxE=Math.max(...sections.map(s=>s.meanEnergy),0);
  sections.forEach((s,i)=>{
    const prev=sections[i-1];
    if(i===0&&s.meanEnergy<0.3&&s.activeBands.length<=2) s.typeGuess='intro';
    else if(prev&&s.activeBands.length>prev.activeBands.length) s.typeGuess='build';
    else if(s.meanEnergy>=maxE*0.95&&s.activeBands.length>=bandEnvs.length-1) s.typeGuess='main';
    else if(prev&&prev.typeGuess==='main'&&s.meanEnergy<prev.meanEnergy*0.6) s.typeGuess='break';
  });
  return sections;
}
```

- [ ] **Step 4: Verify in browser console.**

```js
// noveltyCurve: switching timbre at midpoint produces a peak near the midpoint
const nF=100, nB=12;
const feats=[];
for(let i=0;i<nF;i++){
  const f=new Float32Array(nB);
  if(i<50) f[0]=1; else f[6]=1; // completely different timbre at frame 50
  let norm=0; for(let b=0;b<nB;b++) norm+=f[b]*f[b]; norm=Math.sqrt(norm);
  for(let b=0;b<nB;b++) f[b]/=norm;
  feats.push(f);
}
const nov=noveltyCurve(feats,8);
const peakIdx=nov.indexOf(Math.max(...nov));
console.assert(Math.abs(peakIdx-50)<10,'Expected novelty peak near 50, got '+peakIdx);

// pickBoundaries returns [0, midpoint, duration]
const bounds=pickBoundaries(nov,3,33.3);
console.assert(bounds[0]===0,'First bound should be 0');
console.assert(bounds[bounds.length-1]===33.3,'Last bound should be duration');
console.assert(bounds.length>=3,'Expected at least one internal boundary');

// buildSections: two sections with matching labels for same timbre
const syntheticEnv=new Float32Array(100).fill(0.5);
const bEnvs=[{name:'sub',loHz:30,hiHz:90,env:syntheticEnv},{name:'low',loHz:90,hiHz:200,env:syntheticEnv},
  {name:'lowmid',loHz:200,hiHz:600,env:syntheticEnv},{name:'mid',loHz:600,hiHz:2000,env:syntheticEnv},
  {name:'highmid',loHz:2000,hiHz:6000,env:syntheticEnv},{name:'high',loHz:6000,hiHz:11000,env:syntheticEnv}];
const secs=buildSections([0,11.1,22.2,33.3],bEnvs,43.07,feats,3);
console.assert(secs.length===3,'Expected 3 sections');
console.assert(secs[0].label!==undefined,'Sections should have labels');
console.log('Task 2 assertions passed');
```

Expected: assertions pass, `'Task 2 assertions passed'` logged.

- [ ] **Step 5: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(analysis): SSM-novelty segmentation — coarseFeatures, noveltyCurve, pickBoundaries, buildSections"
```

---

## Task 3: Decode + orchestrator + report

**Files:**
- Modify: `Hypnotize Machine.html` (JS — `decodeToMono22k`, `downsampleEnv`, `lastAnalysis`, `analyzeAudio`)

**Interfaces:**
- Consumes: all Task 1 + Task 2 functions; `ac` (live AudioContext, may be null if not yet initialized)
- Produces:
  - `decodeToMono22k(file:File) → Promise<{data:Float32Array, sampleRate:22050, duration:number}>`
  - `downsampleEnv(env:Float32Array|number[], n?:number) → number[]`
  - `lastAnalysis:object|null` (module-level `let`, starts `null`)
  - `analyzeAudio(file:File, onProgress:(msg:string)=>void) → Promise<object|null>` — returns the full report or `null` on user-cancel
  - Report shape: `{meta:{filename,durationSec,sampleRate,analyzedAt}, tempo, bands:[{name,loHz,hiHz,envelope,entranceSec,peak}], onsets:{envelope,times,frameRate}, sections}`

- [ ] **Step 1: Add `decodeToMono22k` and `downsampleEnv`.** Immediately after `buildSections`:

```js
async function decodeToMono22k(file){
  const buf=await file.arrayBuffer();
  const tmpCtx=new (window.AudioContext||window.webkitAudioContext)();
  let decoded;
  try{ decoded=await tmpCtx.decodeAudioData(buf); }
  finally{ tmpCtx.close(); }
  const duration=decoded.duration;
  const outLen=Math.ceil(duration*22050);
  const offCtx=new OfflineAudioContext(1,outLen,22050);
  const src=offCtx.createBufferSource(); src.buffer=decoded;
  src.connect(offCtx.destination); src.start(0);
  const rendered=await offCtx.startRendering();
  return {data:rendered.getChannelData(0), sampleRate:22050, duration};
}

function downsampleEnv(env, n=800){
  const out=[], step=env.length/n;
  for(let i=0;i<n;i++){
    const s=Math.floor(i*step), e=Math.min(env.length,Math.floor((i+1)*step));
    let sum=0; for(let k=s;k<e;k++) sum+=env[k];
    out.push(e>s?sum/(e-s):0);
  }
  return out;
}
```

- [ ] **Step 2: Add `lastAnalysis` and `analyzeAudio`.** Immediately after:

```js
let lastAnalysis=null;

async function analyzeAudio(file, onProgress){
  onProgress('Decoding…');
  const {data,sampleRate,duration}=await decodeToMono22k(file);
  if(duration>720&&!confirm('This file is '+Math.round(duration/60)+' min long. Analyze anyway?')) return null;

  await new Promise(r=>setTimeout(r,0));
  onProgress('Transform…');
  const {frames,frameRate,freqPerBin}=computeSTFT(data,sampleRate);

  await new Promise(r=>setTimeout(r,0));
  onProgress('Bands & onsets…');
  const bandEnvs=bandEnvelopes(frames,freqPerBin);
  const entrances=detectEntrances(bandEnvs,frameRate);
  const flux=spectralFlux(frames);
  const onsetTimes=pickOnsets(flux,frameRate);

  await new Promise(r=>setTimeout(r,0));
  onProgress('Tempo…');
  const tempo=estimateTempo(flux,frameRate);

  await new Promise(r=>setTimeout(r,0));
  onProgress('Structure…');
  const {features,featRate}=coarseFeatures(frames,freqPerBin,frameRate);
  const novelty=noveltyCurve(features);
  const bounds=pickBoundaries(novelty,featRate,duration);
  const sections=buildSections(bounds,bandEnvs,frameRate,features,featRate);

  onProgress('Done.');
  const report={
    meta:{filename:file.name,durationSec:duration,sampleRate:22050,analyzedAt:new Date().toISOString()},
    tempo,
    bands:bandEnvs.map((b,i)=>({
      name:b.name,loHz:b.loHz,hiHz:b.hiHz,
      envelope:downsampleEnv(b.env),
      entranceSec:entrances[i],
      peak:Math.max(...b.env)
    })),
    onsets:{envelope:downsampleEnv(flux),times:onsetTimes,frameRate},
    sections
  };
  lastAnalysis=report;
  return report;
}
```

- [ ] **Step 3: Verify in browser.** Start `python -m http.server 8765`, open `http://localhost:8765`. Drag a short MP3 or WAV (15–30 s) onto the desktop. In DevTools console:

```js
// Manually drive analyzeAudio to confirm report shape
const input = document.createElement('input');
input.type='file'; input.accept='audio/*';
input.onchange = async e => {
  const file = e.target.files[0];
  const report = await analyzeAudio(file, msg=>console.log(msg));
  console.assert(report.meta.sampleRate===22050, 'sampleRate');
  console.assert(typeof report.tempo.primary==='number', 'tempo.primary');
  console.assert(report.bands.length===6, '6 bands');
  console.assert(report.bands[0].envelope.length===800, 'downsampled to 800');
  console.assert(Array.isArray(report.onsets.times), 'onset times');
  console.assert(Array.isArray(report.sections), 'sections');
  console.log('Report ok, primary BPM:', report.tempo.primary);
  console.log('lastAnalysis set:', lastAnalysis===report);
};
input.click();
```

Expected: progress labels log in sequence (`Decoding… Transform… Bands & onsets… Tempo… Structure… Done.`), assertions pass, BPM printed. No console errors.

- [ ] **Step 4: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(analysis): decode + orchestrator + report (analyzeAudio, lastAnalysis)"
```

---

## Task 4: UI panel + visualization + wiring

**Files:**
- Modify: `Hypnotize Machine.html` (CSS + HTML + JS — panel overlay, canvas draw, readout, button wiring)

**Interfaces:**
- Consumes: `analyzeAudio`, `lastAnalysis`, `downloadBlob`, `drawAnalysis`, `renderReadout`
- Produces:
  - `#analyzeAudio` button in transport
  - `#analyzePanel` overlay with `#analyzeCanvas` and `#analyzeReadout`
  - `drawAnalysis(report:object, canvas:HTMLCanvasElement) → void`
  - `renderReadout(report:object, el:HTMLElement) → void`
  - Export Report JSON button
  - Resize handler re-draws on window resize

- [ ] **Step 1: Add CSS.** In the `<style>` block, after the `.warp-btn.on` line (currently line 207, `  .warp-btn.on{background:#1a3520;border-color:#3a6040;color:#7ac97a;}`), add:

```css
  /* audio analysis panel */
  .analyze-panel{display:none;flex-direction:column;gap:10px;padding:14px 18px;
    border-top:1px solid var(--line);background:rgba(0,0,0,.20);}
  .analyze-panel.show{display:flex;}
  .analyze-head{display:flex;align-items:center;gap:12px;flex-wrap:wrap;}
  .analyze-title{font-size:11px;letter-spacing:.16em;text-transform:uppercase;color:var(--trim);}
  .analyze-tools{display:flex;align-items:center;gap:8px;flex-wrap:wrap;margin-left:auto;}
  .analyze-canvas{width:100%;height:320px;background:var(--lcd-bg);border:1px solid #3a2a12;
    border-radius:8px;display:block;}
  .analyze-readout{font-size:11px;color:var(--ink);letter-spacing:.04em;line-height:1.7;}
  .analyze-readout table{border-collapse:collapse;width:100%;}
  .analyze-readout td,
  .analyze-readout th{padding:2px 8px 2px 0;border-bottom:1px solid var(--line);}
  .analyze-readout .tempo-big{font-size:22px;font-weight:800;color:var(--lcd);}
  .analyze-err{color:#e8552e;font-size:11px;padding:4px 0;}
```

- [ ] **Step 2: Add the `Analyze Audio` button to the transport.** After `<input id="importFile" type="file" accept=".json,application/json" style="display:none">` (currently line 244), before `</div>` that closes `.transport`, add:

```html
        <button id="analyzeAudio" class="btn" title="Analyze an audio file to extract tempo, structure & layers">Analyze Audio</button>
```

- [ ] **Step 3: Add the `#analyzePanel` overlay HTML.** After the `</div>` that closes `#chopEditor` (currently line 300, `    </div>`), before `<!-- song / pattern bar -->` (line 302), add:

```html
    <!-- audio analysis panel -->
    <div id="analyzePanel" class="analyze-panel">
      <div class="analyze-head">
        <span class="analyze-title">Audio Analysis</span>
        <div class="analyze-tools">
          <input id="analyzeFile" type="file" accept="audio/*" style="display:none">
          <span id="analyzeFilename" style="font-size:10px;color:var(--trim-dim);letter-spacing:.06em;"></span>
          <span id="analyzeProgress" style="font-size:10px;color:var(--muted);letter-spacing:.06em;"></span>
          <button id="analyzeExport" class="btn" style="display:none">Export Report JSON</button>
          <button id="analyzeClose" class="btn icon-btn" title="Close">×</button>
        </div>
      </div>
      <canvas id="analyzeCanvas" class="analyze-canvas"></canvas>
      <div id="analyzeReadout" class="analyze-readout"></div>
    </div>
```

- [ ] **Step 4: Add `drawAnalysis`.** In the JS `// === AUDIO ANALYSIS ENGINE ===` block, immediately after `analyzeAudio`:

```js
const BAND_COLORS=['#7055d4','#4a90d9','#45b89c','#f0c040','#e8803a','#e85050'];

function drawAnalysis(report, canvas){
  const w=canvas.width=canvas.clientWidth||800, h=canvas.height=320;
  const ctx=canvas.getContext('2d');
  ctx.clearRect(0,0,w,h);
  const dur=report.meta.durationSec;
  const envH=220, envY=48;

  // stacked band envelopes
  report.bands.forEach((band,bi)=>{
    const env=band.envelope, n=env.length;
    ctx.beginPath(); ctx.moveTo(0,envY+envH);
    for(let i=0;i<n;i++){ const x=i/n*w, y=envY+envH*(1-env[i]); i===0?ctx.moveTo(x,y):ctx.lineTo(x,y); }
    ctx.lineTo(w,envY+envH); ctx.closePath();
    ctx.fillStyle=BAND_COLORS[bi]+'2a'; ctx.fill();
    ctx.strokeStyle=BAND_COLORS[bi]+'bb'; ctx.lineWidth=1.5; ctx.stroke();
  });

  // onset curve
  const oEnv=report.onsets.envelope, oH=30, oY=envY+envH+6;
  ctx.beginPath();
  for(let i=0;i<oEnv.length;i++){ const x=i/oEnv.length*w, y=oY+oH*(1-oEnv[i]); i===0?ctx.moveTo(x,y):ctx.lineTo(x,y); }
  ctx.strokeStyle='rgba(255,255,255,.25)'; ctx.lineWidth=1; ctx.stroke();

  // beat grid
  const beatSec=60/report.tempo.primary;
  ctx.strokeStyle='rgba(255,255,255,.05)'; ctx.lineWidth=1;
  for(let t=0;t<dur;t+=beatSec){ const x=t/dur*w; ctx.beginPath(); ctx.moveTo(x,envY); ctx.lineTo(x,envY+envH); ctx.stroke(); }

  // section dividers + labels
  ctx.setLineDash([]);
  report.sections.forEach((sec,i)=>{
    if(i===0) return;
    const x=sec.startSec/dur*w;
    ctx.strokeStyle='rgba(255,255,255,.2)'; ctx.lineWidth=1;
    ctx.beginPath(); ctx.moveTo(x,envY); ctx.lineTo(x,envY+envH); ctx.stroke();
    ctx.fillStyle='rgba(255,255,255,.5)'; ctx.font='9px monospace';
    ctx.fillText(sec.label+'·'+sec.typeGuess, x+2, envY+10);
  });

  // entrance markers
  report.bands.forEach((band,bi)=>{
    if(band.entranceSec==null) return;
    const x=band.entranceSec/dur*w;
    ctx.strokeStyle=BAND_COLORS[bi]; ctx.lineWidth=1.5;
    ctx.setLineDash([4,3]);
    ctx.beginPath(); ctx.moveTo(x,envY); ctx.lineTo(x,envY+envH); ctx.stroke();
    ctx.setLineDash([]);
    ctx.fillStyle=BAND_COLORS[bi]; ctx.font='9px monospace';
    ctx.fillText(band.name, x+2, envY+14+bi*11);
  });

  // band legend left edge
  report.bands.forEach((band,bi)=>{
    ctx.fillStyle=BAND_COLORS[bi]; ctx.font='9px monospace';
    ctx.fillText(band.name, 2, envY+envH-(bi*14)-4);
  });

  // time axis
  const step=Math.ceil(dur/8);
  ctx.fillStyle='rgba(255,255,255,.3)'; ctx.font='9px monospace';
  for(let t=0;t<=dur;t+=step){
    const x=Math.min(w-24,t/dur*w);
    ctx.fillText(String(Math.floor(t/60))+':'+String(t%60).padStart(2,'0'), x, envY-4);
  }
}
```

- [ ] **Step 5: Add `renderReadout`.** Immediately after `drawAnalysis`:

```js
function renderReadout(report, el){
  const t=report.tempo;
  const fmt=s=>String(Math.floor(s/60))+':'+String(Math.floor(s%60)).padStart(2,'0');
  let h=`<div style="margin-bottom:10px">
    <span class="tempo-big">${t.primary} BPM</span>
    <span style="color:var(--muted);font-size:10px;margin-left:10px;">½ = ${t.halfTime} · 2× = ${t.doubleTime}</span>
    <div style="margin-top:4px;display:flex;gap:6px;flex-wrap:wrap;">
      ${(t.candidates||[]).slice(0,6).map(c=>`<span style="background:var(--recess);border:1px solid var(--line);border-radius:4px;padding:2px 6px;font-size:10px;">${c.bpm}<span style="color:var(--muted);"> ${Math.round(c.strength*100)}%</span></span>`).join('')}
    </div></div>
  <table style="margin-bottom:10px">
    <tr><th style="text-align:left;color:var(--muted);font-weight:normal;">Band</th><th style="text-align:left;color:var(--muted);font-weight:normal;">Enters</th><th style="text-align:left;color:var(--muted);font-weight:normal;">Active</th></tr>
    ${report.bands.map(b=>`<tr><td>${b.name}</td><td>${b.entranceSec!=null?fmt(b.entranceSec):'—'}</td><td>${b.entranceSec!=null?'✓':'✗'}</td></tr>`).join('')}
  </table>
  <div style="color:var(--muted);font-size:10px;letter-spacing:.1em;text-transform:uppercase;margin-bottom:4px;">Sections</div>
  <table>
    ${report.sections.map(s=>`<tr>
      <td style="font-weight:800;color:var(--trim);padding-right:10px">${s.label}</td>
      <td>${fmt(s.startSec)}–${fmt(s.endSec)}</td>
      <td style="color:var(--muted);padding:0 10px">${s.typeGuess}</td>
      <td style="color:var(--muted);font-size:10px">${s.activeBands.join(', ')}</td>
    </tr>`).join('')}
  </table>`;
  if(report.onsets.times.length===0) h+=`<div class="analyze-err" style="margin-top:8px">No clear rhythmic content detected.</div>`;
  el.innerHTML=h;
}
```

- [ ] **Step 6: Add panel wiring JS.** Immediately after `renderReadout`:

```js
// --- audio analysis panel ---
const analyzePanel=document.getElementById('analyzePanel');
const analyzeCanvas=document.getElementById('analyzeCanvas');
const analyzeReadout=document.getElementById('analyzeReadout');
const analyzeProgress=document.getElementById('analyzeProgress');
const analyzeFilename=document.getElementById('analyzeFilename');
const analyzeExportBtn=document.getElementById('analyzeExport');

document.getElementById('analyzeAudio').onclick=()=>{
  const wasOpen=analyzePanel.classList.contains('show');
  analyzePanel.classList.toggle('show');
  if(!wasOpen) document.getElementById('analyzeFile').click();
};
document.getElementById('analyzeClose').onclick=()=>analyzePanel.classList.remove('show');

document.getElementById('analyzeFile').onchange=async(e)=>{
  const file=e.target.files[0]; if(!file) return;
  analyzeFilename.textContent=file.name;
  analyzeReadout.innerHTML='';
  analyzeExportBtn.style.display='none';
  try{
    const report=await analyzeAudio(file,msg=>{analyzeProgress.textContent=msg;});
    if(!report) return;
    drawAnalysis(report,analyzeCanvas);
    renderReadout(report,analyzeReadout);
    analyzeExportBtn.style.display='';
  }catch(err){
    analyzeReadout.innerHTML=`<div class="analyze-err">Could not decode this file: ${err.message}</div>`;
    analyzeProgress.textContent='';
  }
  e.target.value='';
};

analyzeExportBtn.onclick=()=>{
  if(!lastAnalysis) return;
  const fname=(lastAnalysis.meta.filename||'analysis').replace(/\.[^.]+$/,'')+'.analysis.json';
  downloadBlob(new Blob([JSON.stringify(lastAnalysis,null,2)],{type:'application/json'}),fname);
};

window.addEventListener('resize',()=>{
  if(analyzePanel.classList.contains('show')&&lastAnalysis) drawAnalysis(lastAnalysis,analyzeCanvas);
});
```

- [ ] **Step 7: Verify UI in browser.** Reload `http://localhost:8765`. Run through the QA checklist:

  1. Click **Analyze Audio** → panel opens + file picker opens. Select a real audio file (MP3/WAV/FLAC). Confirm:
     - Progress labels cycle: `Decoding… Transform… Bands & onsets… Tempo… Structure… Done.`
     - Canvas renders: 6 colored band envelopes, onset strip at bottom, vertical entrance markers, section dividers with labels.
     - Readout shows: tempo big number, ½ / 2× chips, candidate BPM chips, band entrance table, sections list.
  2. Click **Export Report JSON** → a `.analysis.json` downloads. Inspect it matches the schema (`meta`, `tempo`, `bands[0].envelope.length===800`, `sections` array).
  3. Resize the window → canvas redraws automatically.
  4. Click × → panel closes.
  5. Reopen (Analyze Audio again) → panel opens without triggering file picker a second time.
  6. Feed a deliberately unsupported file (e.g., a `.txt` renamed to `.mp3`) → `analyze-err` message appears, no crash.
  7. No JS console errors throughout.

- [ ] **Step 8: Commit.**

```bash
git add "Hypnotize Machine.html"
git commit -m "feat(analysis): UI panel, canvas chart, readout, export — Piece 2 complete"
```

---

## Self-review against the spec

**Spec coverage:**
- §3 Decode & STFT: `decodeToMono22k` (OfflineAudioContext 1ch 22050 Hz), `fft` (in-place radix-2), `computeSTFT` (Hann window, frameSize=1024, hop=512) → Task 1. ✓
- §4 Band energies & entrances: `BANDS` (6 named ranges), `bandEnvelopes` (smooth + normalize), `detectEntrances` (sustain threshold) → Task 1. ✓
- §5 Onsets & tempo: `spectralFlux`, `pickOnsets` (adaptive threshold), `estimateTempo` (ACF + prior + harmonic family) → Task 1. ✓
- §6 SSM-novelty: `coarseFeatures` (12 log-spaced, L2-norm), `noveltyCurve` (checkerboard kernel, Gaussian-tapered, no full N×N stored), `pickBoundaries`, `buildSections` (greedy letter labels, typeGuess post-pass) → Task 2. ✓
- §7 Report structure: exact shape with `meta`, `tempo`, `bands[].envelope` (800-pt downsampled), `onsets`, `sections` → Task 3. ✓
- §7 `lastAnalysis` global → Task 3. ✓
- §7 `downsampleEnv` → exact values never downsampled (tempo, entranceSec, onsets.times, section bounds). ✓
- §8 UI: `#analyzePanel` overlay (`.show` toggle), progress label, canvas, readout, Export JSON button, resize handler, close button → Task 4. ✓
- §8 "Analyze Audio" button in header transport → Task 4. ✓
- §9 >720s confirm dialog → Task 3. ✓
- §9 Decode error → inline error message → Task 4. ✓
- §9 Empty/silent → report builds, `No clear rhythmic content` note → Task 4 `renderReadout`. ✓
- §10 QA: synthetic console assertions (FFT peak, 120 BPM recovery, entrance detection, band separation, novelty peak) → Tasks 1 & 2 verify steps. ✓

**Placeholder scan:** No TBD/TODO. All steps contain complete code. ✓

**Type consistency:** `bandEnvs` passed from `bandEnvelopes` → `detectEntrances` → `buildSections` as `[{name,loHz,hiHz,env:Float32Array}]` — consistent throughout. `frames:Float32Array[]`, `frameRate:number`, `freqPerBin:number` consistent. `features:Float32Array[]`, `featRate:number` consistent between `coarseFeatures` → `noveltyCurve` → `pickBoundaries` → `buildSections`. ✓
