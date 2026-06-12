# PlayAlong Practice Features — Design

**Date:** 2026-06-12
**Scope:** Three practice-oriented features for the PlayAlong guitar tab player, plus
repo/feature-tracking setup. Everything stays in the single static `index.html`
(zero dependencies, Web Audio API).

## Context

PlayAlong already has: a step timeline built from parsed ASCII tab
(`rebuildTimeline`), a look-ahead Web Audio scheduler (`schedulerTick`) that wraps
playback at `loopBounds()`, a count-in click sound (`playTick`), whole-song and
single-part looping (chip UI + shaded `loopRegion` on the progress bar), a tempo
slider with percent-of-written-tempo readout, and localStorage persistence for
user tabs. The three features below are deliberate extensions of that machinery,
not new subsystems.

## Feature 1: Speed trainer (progressive tempo)

**Purpose:** Practice a passage slowly and let the app raise the tempo
automatically as you repeat it.

- A **Trainer** toggle button in the tempo control group, with a small popover
  containing three number inputs:
  - **Start** — percent of written tempo, default **60%**
  - **Step** — percent added per loop pass, default **+5%**
  - **Target** — percent to stop at, default **100%**
- When the trainer is armed and playback starts, the tempo is set to
  `start% × written tempo`.
- Each time playback wraps at the loop boundary (the existing loop-back branch in
  `schedulerTick`), tempo increases by `step%` until it reaches `target%`, then
  holds there.
- Works with any loop kind: whole song, part loop, or A/B selection (feature 2).
- The existing `tempoVal` readout (BPM + %) shows progress; no new readout needed.
- Disabling the trainer restores the tempo the user had set manually before arming.
- Trainer settings persist in localStorage; the armed/disarmed state resets per
  page load.

**Edge cases:** manual tempo changes while the trainer is armed re-base the ramp
(trainer continues stepping from the new value). If loop is off entirely, the
trainer arms but does nothing until a loop wrap occurs (song-end restart counts
as a wrap when whole-song loop is on).

## Feature 2: A/B selection loop

**Purpose:** Loop an arbitrary phrase (e.g. two bars), not just a whole part.

- **Drag across columns** in the tab to select a range: `mousedown` on a column +
  drag highlights the covered columns; `mouseup` sets the loop to that range.
- A plain click without crossing into another column behaves exactly as today
  (seek to that step).
- The selection is shown three ways, all reusing existing UI:
  - selected columns get a tinted background in the tab,
  - the `loopRegion` stripe covers the range on the progress bar,
  - the `loopChip` shows e.g. `Selection · m. 3–4` with the existing ✕ to clear.
- **One loop at a time:** whole-song loop, part loop, and selection loop are
  mutually exclusive. Setting one clears the others. All playback decisions keep
  flowing through the single `loopBounds()` function, which gains a selection
  branch.
- Selection survives pause/stop but not switching songs.

## Feature 3: Metronome click + silent guitar mode

**Purpose:** Hear a click while playing along; optionally mute the synthesized
guitar so the app acts as a conductor (focus bar + click only) while the learner
plays the real notes.

- Two independent toggle buttons in the transport:
  - **Click** — schedules a tick on every beat (`step % subdivision === 0`)
    alongside note playback, accented on measure starts. Reuses the count-in's
    `playTick` sound.
  - **Mute guitar** — routes all plucked-string output through a new guitar-bus
    `GainNode`; muting sets that bus to 0. Notes are still scheduled, so the
    focus bar, progress, and position readout behave identically.
- Click volume follows the master volume slider (same as count-in today).
- Both toggle states persist in localStorage like other preferences.
- "Silent guitar practice" is simply Click on + Mute guitar on — no third mode.

## Repo & feature tracking

- Git repo in `playalong/`, initial commit of current state, branch `main`.
- Repo-local git identity `styxx3542 <96124546+styxx3542@users.noreply.github.com>`
  (personal account; the work account is never used).
- Public GitHub repo `styxx3542/playalong`, pushed via the personal `gh` account.
- **Feature tracking via GitHub Issues:** one issue per feature above; each
  feature lands as its own commit on `main` with `Closes #N` in the message.

## Testing

Manual, in-browser (Web Audio app, no test harness):

1. Trainer: arm with 60/+5/100 on a part loop; verify tempo readout climbs 5%
   per pass and stops at 100%; disarm restores prior tempo.
2. Selection: drag two bars; verify loop chip, progress stripe, column tint,
   playback wrapping at the right columns; plain click still seeks; setting a
   part loop clears the selection and vice versa.
3. Metronome: click aligns with beats at multiple tempos and subdivisions;
   accent on measure starts; mute silences guitar while focus bar advances;
   states survive reload.
