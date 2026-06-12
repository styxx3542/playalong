# PlayAlong Practice Features Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three practice features to the PlayAlong tab player — metronome click + guitar mute (issue #3), A/B selection loop (issue #2), speed trainer (issue #1) — each as its own commit on `main` closing its issue.

**Architecture:** Everything lives in the single static `index.html` (CSS block, HTML transport markup, and the `<script>` at the bottom). Each feature extends existing machinery: the metronome reuses `playTick` inside `schedulerTick`; the mute inserts a `guitarBus` GainNode between note gains and `master`; the selection loop adds a `state.loopSel` branch to `loopBounds()` plus drag handlers on the already-rendered column divs; the trainer hooks the loop-wrap branch of `schedulerTick`.

**Tech Stack:** Vanilla JS, Web Audio API, no dependencies. **No automated test harness exists and we are not adding one** (the app is one static file; its behavior is audio + DOM timing). Every task ends with a manual in-browser verification checklist instead of unit tests. Serve with `python3 -m http.server 8765` from the repo root and open `http://localhost:8765`.

**Implementation order:** Task 1 = issue #3 (smallest, independent), Task 2 = issue #2, Task 3 = issue #1 (trainer benefits from selection loops existing first), Task 4 = README + push. Line numbers below refer to the current `index.html` at commit `ba0aac3`; they shift as tasks land, so locate edits by the quoted anchor code, not the number.

---

### Task 1: Metronome click + guitar mute bus (Closes #3)

**Files:**
- Modify: `index.html` (HTML transport row ~line 286-290, `ensureAudio` ~line 517, `playNote` ~line 609, `playMute` ~line 625, `state` ~line 631, `schedulerTick` ~line 715, wiring section ~line 1288)

- [ ] **Step 1: Add the two toggle buttons to the transport**

In the loop `ctl-group` (the one containing `loopBtn` / `loopChip` / `countBtn`), after the `countBtn` line:

```html
          <button class="ctl" id="metroBtn" title="Metronome click on every beat during playback">Click</button>
          <button class="ctl" id="muteBtn" title="Mute the guitar sound — focus bar and click keep going">Mute 🎸</button>
```

- [ ] **Step 2: Add state fields**

In the `const state = {...}` object, after the `countIn:` line:

```js
  metro: localStorage.getItem("pa.metro") === "1",
  muteGuitar: localStorage.getItem("pa.muteGuitar") === "1",
```

(Both default off for new users.)

- [ ] **Step 3: Create the guitar bus in the audio graph**

Below `let master = null;` add:

```js
let guitarBus = null;
```

Inside `ensureAudio()`, right after `master.connect(body); body.connect(shelf); shelf.connect(comp);`:

```js
    guitarBus = ctx.createGain();
    guitarBus.gain.value = state.muteGuitar ? 0 : 1;
    guitarBus.connect(master);
```

- [ ] **Step 4: Route guitar voices through the bus**

In `playNote`, change `g.connect(master);` to:

```js
  g.connect(guitarBus);
```

In `playMute`, change `src.connect(f); f.connect(g); g.connect(master);` to:

```js
  src.connect(f); f.connect(g); g.connect(guitarBus);
```

`playTick` (count-in + metronome) stays connected to `master` so the click is audible while the guitar is muted.

- [ ] **Step 5: Schedule the click in the scheduler**

In `schedulerTick`, after `const entry = state.timeline[state.stepIdx];` and its `if (!entry)` guard, before `scheduleStep(entry, state.nextNoteTime);`:

```js
    if (state.metro && state.measureStarts.length) {
      const m = measureOfStep(state.stepIdx);
      const off = state.stepIdx - state.measureStarts[m];
      if (off % state.song.subdivision === 0) playTick(state.nextNoteTime, off === 0);
    }
```

(Beat = every `subdivision` steps from the measure start; accent on the measure's first step — same sound as the count-in.)

- [ ] **Step 6: Wire the toggles with persistence**

In the wiring section, after the `countBtn` listener block:

```js
$("metroBtn").classList.toggle("toggled", state.metro);
$("metroBtn").addEventListener("click", e => {
  state.metro = !state.metro;
  localStorage.setItem("pa.metro", state.metro ? "1" : "0");
  e.currentTarget.classList.toggle("toggled", state.metro);
});

$("muteBtn").classList.toggle("toggled", state.muteGuitar);
$("muteBtn").addEventListener("click", e => {
  state.muteGuitar = !state.muteGuitar;
  localStorage.setItem("pa.muteGuitar", state.muteGuitar ? "1" : "0");
  e.currentTarget.classList.toggle("toggled", state.muteGuitar);
  if (guitarBus) guitarBus.gain.setTargetAtTime(state.muteGuitar ? 0 : 1, ctx.currentTime, 0.01);
});
```

(`setTargetAtTime` avoids a click when muting mid-note.)

- [ ] **Step 7: Verify in browser**

Run: `python3 -m http.server 8765` (repo root), open `http://localhost:8765`, load "Ode to Joy".

- Click toggles on → play: tick on every beat, higher-pitched accent at each measure start, aligned with notes at 120 BPM and after dropping tempo to 60 BPM.
- Mute 🎸 on → play: no guitar, focus bar + position readout still advance, click still audible. Toggle mute mid-playback: guitar cuts/returns without a pop.
- Count-in still works and is audible with mute on.
- Reload page: both toggle states restored.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add metronome click and guitar mute for silent practice

Closes #3

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: A/B selection loop via drag (Closes #2)

**Files:**
- Modify: `index.html` (CSS `.block`/`.col` rules ~line 125-146, `state` ~line 631, `loopBounds` ~line 703, `schedulerTick` ~line 720, `renderSong` column wiring ~line 863-867, `toggleLoopPart` ~line 1067, `updateLoopUI` ~line 1087, `afterStructureChange` ~line 1115, `loadSong` ~line 1229, `loopBtn`/`loopChipClear` wiring ~line 1281-1286)

- [ ] **Step 1: CSS for the selection tint + disable text selection while dragging**

Add `user-select: none;` to the existing `.block` rule, and add a new rule after `.col.playhead .c.note`:

```css
  .col.selrange { background: rgba(245,165,36,0.10); box-shadow: inset 0 -2px 0 rgba(245,165,36,0.5); }
```

- [ ] **Step 2: Add state field**

In `const state = {...}`, after `loopPartId: null,`:

```js
  loopSel: null,      // {lo, hi} half-open timeline step range, or null
```

- [ ] **Step 3: Selection branch in `loopBounds` and the scheduler's wrap condition**

At the top of `loopBounds()`, before the `if (state.loopPartId)` branch:

```js
  if (state.loopSel) {
    const lo = Math.max(0, state.loopSel.lo);
    const hi = Math.min(state.timeline.length, state.loopSel.hi);
    if (lo < hi) return [lo, hi];
  }
```

In `schedulerTick`, change `if (state.loopAll || state.loopPartId) {` to:

```js
      if (state.loopAll || state.loopPartId || state.loopSel) {
```

- [ ] **Step 4: Selection set/clear/paint helpers**

After `clearPartLoop()` (~line 1085), add:

```js
function paintSelection() {
  const sel = state.loopSel;
  colEls.forEach((el, i) => {
    if (el) el.classList.toggle("selrange", !!sel && i >= sel.lo && i < sel.hi);
  });
}

function setSelectionLoop(lo, hi) {
  state.loopPartId = null;
  state.loopSel = { lo, hi };
  paintSelection();
  renderParts();          // drop any part-loop highlight
  updateLoopUI();
  seekTo(lo);
}

function clearSelectionLoop() {
  if (!state.loopSel) return;
  state.loopSel = null;
  paintSelection();
  updateLoopUI();
}
```

- [ ] **Step 5: Drag interaction on the rendered columns**

Above `renderSong` (next to the `let colEls = [];` declarations), add:

```js
let dragSel = null;            // {anchor, last} while the mouse is down on a column
let suppressNextClick = false; // a completed drag must not also fire the click-to-seek
```

In `renderSong`, replace the existing column wiring:

```js
        if (!col.isBar && part.enabled) {
          const myIdx = timelineIdx++;
          colEls[myIdx] = colEl;
          colEl.addEventListener("click", () => seekTo(myIdx));
        }
```

with:

```js
        if (!col.isBar && part.enabled) {
          const myIdx = timelineIdx++;
          colEls[myIdx] = colEl;
          colEl.addEventListener("click", () => {
            if (suppressNextClick) { suppressNextClick = false; return; }
            seekTo(myIdx);
          });
          colEl.addEventListener("mousedown", e => {
            if (e.button !== 0) return;
            e.preventDefault();
            dragSel = { anchor: myIdx, last: myIdx };
          });
          colEl.addEventListener("mouseover", () => {
            if (!dragSel) return;
            dragSel.last = myIdx;
            const lo = Math.min(dragSel.anchor, dragSel.last);
            const hi = Math.max(dragSel.anchor, dragSel.last);
            colEls.forEach((el, i) => { if (el) el.classList.toggle("selrange", i >= lo && i <= hi); });
          });
        }
```

In the wiring section at the bottom, add the global mouseup that commits the drag:

```js
document.addEventListener("mouseup", () => {
  if (!dragSel) return;
  const { anchor, last } = dragSel;
  dragSel = null;
  if (anchor === last) { paintSelection(); return; } // plain click: let click-to-seek handle it
  suppressNextClick = true;
  setSelectionLoop(Math.min(anchor, last), Math.max(anchor, last) + 1);
});
```

At the end of `renderSong`, after `highlightStep(...)`, add `paintSelection();` so re-renders keep the tint.

- [ ] **Step 6: Mutual exclusivity + chip/region UI**

In `toggleLoopPart`, after `state.loopPartId = state.loopPartId === pid ? null : pid;` add:

```js
  if (state.loopPartId) { state.loopSel = null; }
```

Replace the body of `updateLoopUI` with:

```js
function updateLoopUI() {
  const chip = $("loopChip");
  const region = $("loopRegion");
  const total = state.timeline.length || 1;
  let label = null, lo = 0, hi = 0;
  if (state.loopSel && state.song) {
    [lo, hi] = loopBounds();
    const m1 = measureOfStep(lo) + 1, m2 = measureOfStep(Math.max(lo, hi - 1)) + 1;
    label = "Looping: " + (m1 === m2 ? "measure " + m1 : "measures " + m1 + "–" + m2);
  } else if (state.loopPartId && state.song) {
    const part = state.song.parts.find(p => p.id === state.loopPartId);
    label = "Looping: " + (part ? part.name : "?");
    [lo, hi] = loopBounds();
  }
  if (label) {
    $("loopChipText").textContent = label;
    chip.style.display = "inline-flex";
    region.style.left = (lo / total * 100) + "%";
    region.style.width = ((hi - lo) / total * 100) + "%";
    region.style.display = "block";
  } else {
    chip.style.display = "none";
    region.style.display = "none";
  }
}
```

Update the two clear paths in the wiring section:

```js
$("loopBtn").addEventListener("click", () => {
  if (state.loopSel) { clearSelectionLoop(); toast("Selection loop released"); return; }
  if (state.loopPartId) { clearPartLoop(); toast("Part loop released"); return; }
  state.loopAll = !state.loopAll;
  $("loopBtn").classList.toggle("toggled", state.loopAll);
});
$("loopChipClear").addEventListener("click", () => { clearSelectionLoop(); clearPartLoop(); });
```

- [ ] **Step 7: Invalidate the selection when step indexes change**

In `afterStructureChange`, immediately before `rebuildTimeline();`:

```js
  state.loopSel = null;   // timeline indexes shift when parts move/skip
```

In `loadSong`, next to `state.loopPartId = null;`:

```js
  state.loopSel = null;
```

- [ ] **Step 8: Verify in browser**

Load "Greensleeves" (multi-part song):

- Drag across ~2 bars: columns tint while dragging; on release the chip reads "Looping: measures X–Y", the striped region appears on the progress bar, playback starts from the selection start and wraps inside it.
- Plain click (no drag) on a column still seeks; no chip appears.
- With a selection active, set a part loop (⟳ on a part): selection clears, part loop takes over. Drag a new selection: part loop clears.
- ✕ on the chip clears whichever loop is active; Loop ⟳ button also releases it (toast confirms).
- Pause/stop: selection survives. Reorder or skip a part: selection clears. Switch songs: selection clears.
- Selection + count-in + metronome together behave sanely.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Add A/B selection loop: drag across tab columns to loop a phrase

Closes #2

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: Speed trainer (Closes #1)

**Files:**
- Modify: `index.html` (CSS ~line 246, transport HTML tempo group ~line 284, transport container, `state` ~line 631, `schedulerTick` wrap branch, `loadSong`, wiring section)

- [ ] **Step 1: Trainer button + popover markup**

In the tempo `ctl-group`, after the `tempoReset` button:

```html
          <button class="ctl" id="trainerBtn" title="Speed trainer: raise the tempo every loop pass">Trainer</button>
```

Just before `<div id="toast"></div>` near the end of `<body>`:

```html
<div id="trainerPop">
  <label>Start <span><input type="number" id="trStart" min="20" max="100" step="5"> %</span></label>
  <label>Step <span>+<input type="number" id="trStep" min="1" max="50" step="1"> %</span></label>
  <label>Target <span><input type="number" id="trTarget" min="30" max="200" step="5"> %</span></label>
  <button class="ctl" id="trArm">Start training</button>
  <p class="hint">Starts at Start% of the written tempo and rises by Step% each time the loop wraps, holding at Target%.</p>
</div>
```

- [ ] **Step 2: Popover CSS**

After the `#toast` rule:

```css
  #trainerPop {
    display: none; position: fixed; z-index: 45; width: 240px;
    background: var(--panel-2); border: 1px solid var(--border); border-radius: 8px;
    padding: 12px; box-shadow: 0 6px 24px rgba(0,0,0,0.45);
  }
  #trainerPop.open { display: block; }
  #trainerPop label {
    display: flex; align-items: center; justify-content: space-between;
    font-size: 12px; color: var(--muted); margin-bottom: 8px;
  }
  #trainerPop input {
    width: 60px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 4px; padding: 4px 6px; font-size: 13px;
  }
  #trainerPop > button { width: 100%; margin-top: 2px; }
```

- [ ] **Step 3: State + persisted settings**

In `const state = {...}`, after `fontSize:`:

```js
  trainer: { armed: false, start: 60, step: 5, target: 100, savedBpm: null },
```

Right after the `const state = {...};` declaration closes:

```js
try {
  const t = JSON.parse(localStorage.getItem("pa.trainer") || "{}");
  if (+t.start) state.trainer.start = Math.max(20, Math.min(100, +t.start));
  if (+t.step) state.trainer.step = Math.max(1, Math.min(50, +t.step));
  if (+t.target) state.trainer.target = Math.max(30, Math.min(200, +t.target));
} catch (e) {}
```

(Armed state deliberately not persisted — it resets per page load, per spec.)

- [ ] **Step 4: Trainer logic**

In the `/* ---------- Tempo ---------- */` section after `syncTempoUI`:

```js
/* ---------- Speed trainer ---------- */

function saveTrainerPrefs() {
  localStorage.setItem("pa.trainer", JSON.stringify({
    start: state.trainer.start, step: state.trainer.step, target: state.trainer.target,
  }));
}

function syncTrainerUI() {
  $("trStart").value = state.trainer.start;
  $("trStep").value = state.trainer.step;
  $("trTarget").value = state.trainer.target;
  $("trArm").textContent = state.trainer.armed ? "Stop training" : "Start training";
  $("trainerBtn").classList.toggle("toggled", state.trainer.armed);
}

function armTrainer() {
  if (!state.song) { toast("Load a tab first"); return; }
  state.trainer.armed = true;
  state.trainer.savedBpm = state.bpm;
  setBpm(state.song.tempo * state.trainer.start / 100);
  syncTrainerUI();
  toast(`Trainer on: ${state.trainer.start}% → ${state.trainer.target}%, +${state.trainer.step}% per pass`);
}

function disarmTrainer() {
  state.trainer.armed = false;
  if (state.trainer.savedBpm) setBpm(state.trainer.savedBpm);
  state.trainer.savedBpm = null;
  syncTrainerUI();
}

// Called from schedulerTick each time playback wraps at the loop end.
// Computes the ramp from the CURRENT bpm, so manual slider moves re-base it.
function trainerOnWrap() {
  if (!state.trainer.armed || !state.song) return;
  const pct = state.bpm / state.song.tempo * 100;
  if (pct >= state.trainer.target - 0.5) return;
  setBpm(state.song.tempo * Math.min(state.trainer.target, pct + state.trainer.step) / 100);
}
```

- [ ] **Step 5: Hook the loop wrap**

In `schedulerTick`, replace the wrap branch body:

```js
    if (state.stepIdx >= hi || state.stepIdx < lo) {
      if (state.loopAll || state.loopPartId || state.loopSel) {
        const wrapped = state.stepIdx >= hi;
        state.stepIdx = lo;
        if (wrapped) trainerOnWrap();
      } else {
        finishPlayback();
        return;
      }
    }
```

(Only an end-of-loop wrap advances the ramp; a backwards seek that lands below `lo` does not count as a completed pass.)

- [ ] **Step 6: Disarm on song switch**

In `loadSong`, after `state.bpm = song.tempo;`:

```js
  state.trainer.armed = false;
  state.trainer.savedBpm = null;
```

And after the existing `syncTempoUI();` call in `loadSong`, add `syncTrainerUI();`.

- [ ] **Step 7: Wire the popover**

In the wiring section, after the tempo listeners:

```js
$("trainerBtn").addEventListener("click", () => {
  const pop = $("trainerPop");
  if (pop.classList.contains("open")) { pop.classList.remove("open"); return; }
  syncTrainerUI();
  const r = $("trainerBtn").getBoundingClientRect();
  pop.style.left = Math.max(8, Math.min(window.innerWidth - 256, r.left - 20)) + "px";
  pop.style.bottom = (window.innerHeight - r.top + 8) + "px";
  pop.classList.add("open");
});
document.addEventListener("mousedown", e => {
  const pop = $("trainerPop");
  if (pop.classList.contains("open") && !pop.contains(e.target) &&
      e.target !== $("trainerBtn")) pop.classList.remove("open");
});
$("trArm").addEventListener("click", () => {
  state.trainer.armed ? disarmTrainer() : armTrainer();
});
$("trStart").addEventListener("change", e => {
  state.trainer.start = Math.max(20, Math.min(100, +e.target.value || 60));
  e.target.value = state.trainer.start; saveTrainerPrefs();
});
$("trStep").addEventListener("change", e => {
  state.trainer.step = Math.max(1, Math.min(50, +e.target.value || 5));
  e.target.value = state.trainer.step; saveTrainerPrefs();
});
$("trTarget").addEventListener("change", e => {
  state.trainer.target = Math.max(30, Math.min(200, +e.target.value || 100));
  e.target.value = state.trainer.target; saveTrainerPrefs();
});
syncTrainerUI();
```

(The existing keydown handler already ignores number inputs — `t.type !== "range"` — so typing in the popover won't trigger Space/arrows.)

- [ ] **Step 8: Verify in browser**

Load "Twelve Bar Blues", select ~2 bars (Task 2's drag):

- Open Trainer popover, defaults 60 / +5 / 100, click "Start training": tempo readout drops to 60% of written tempo, button highlights.
- Play: each loop pass raises tempo +5% (watch the readout climb), stopping at 100% and holding.
- Mid-ramp, drag the tempo slider down: ramp continues from the new value.
- "Stop training": tempo restores to what it was before arming.
- Works the same with a part loop and with whole-song Loop ⟳ (wrap at song end counts as a pass).
- Settings changes survive reload; armed state does not.
- Pause/resume mid-ramp does NOT reset to start% (ramp continues).

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Add speed trainer: progressive tempo on each loop pass

Closes #1

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: README + push

**Files:**
- Modify: `README.md` (Features section)

- [ ] **Step 1: Document the new features**

In `README.md`'s Features list, after the **Count-in** bullet:

```markdown
- **Metronome** — click on every beat during playback (accented on measure starts), toggleable
- **Silent guitar mode** — mute the synthesized guitar and play along with just the click and focus bar
- **A/B selection loop** — drag across the tab to loop any phrase; chip + progress-bar stripe show the region
- **Speed trainer** — start a loop at e.g. 60% tempo and let it rise +5% every pass until 100%
```

- [ ] **Step 2: Commit and push**

```bash
git add README.md
git commit -m "Document practice features in README

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push origin main
```

Expected: push succeeds; issues #1, #2, #3 auto-close on GitHub from the commit messages.

- [ ] **Step 3: Final smoke test**

With the server still running, hard-reload the page and run one combined pass: metronome on, guitar muted, 2-bar selection, trainer 60→100: verify the tempo climbs each pass while the click stays aligned and the chip/region stay put. Verify no console errors.
