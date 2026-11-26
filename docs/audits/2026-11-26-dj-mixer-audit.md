# DJ mixer audit — `drop-ins/dj-mixer-01` (2026-11-26)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-26

Scope: the whole `dj-mixer-01` drop-in at submodule commit `1012c82`
(`0.222.3`, 38 modules, 35,318 lines, plus 29,204 lines of tests), in three
parts:

- **Engine** — `mixer_engine.py`, `deck.py`, `dsp.py`, `stems.py`,
  `sampler.py`, `audio_process.py`, `loop_export.py`, and the analysis modules
  (`analysis_job.py`, `bpm.py`, `key_detect.py`, `structure.py`).
- **UI** — `ui.py` (13,344 lines).
- **Control, library, hardware** — `dj_mixer_controller.py`, `autoplay.py`,
  the browser, library, track store, sets, tags and file operations, and the
  REV1, REV1000, S4 MK3 and XDJ-XZ input and LED modules.

**Tuning was out of scope.**  No BPM, key or structure threshold is judged.

Method:

- Every line of every module was read.  The REV1000 files were read as a diff
  against their REV1 originals (they differ in names and comments only).  The
  drop-in's docs were checked for key names and versions, not read as prose.
- The engine was exercised numerically: block-render timing for four
  synthetic decks, and signal checks (unity gain, fader steps, loops, keylock,
  resampling, NaN input) against known signals.  No audio device was opened;
  `sounddevice` was stubbed where `start()` was tested.
- The UI was driven headless with a real engine object (never started), real
  decks with synthetic audio, a real browser and store on scratch folders, and
  synthetic SDL events.
- Library, store and file-operation code ran only against scratch files with
  `UNICORNVIZ_APP_ROOT` pointed at a scratch directory.  The live checkout and
  the owner's music library were never read or written.
- No MIDI port or HID device was opened; decoders ran on synthetic bytes.
  Demucs was not run.  Nothing was tested on Windows.
- **VERIFIED** = reproduced by script; the most severe finding of each part
  was re-run by a second reader.  **SUSPECTED** marks what could not be
  confirmed.

Repro scripts: `/var/tmp/uv-mixer-audit/{engine,ui,ctrl,mine}/`.

Severity: **P1** an audible dropout or silent output, a crash, hang or
stalled set, an unintended action on a playing deck, or the owner's files or
data damaged.  **P2** audibly wrong output in a specific situation, a control
that misbehaves, data silently wrong, or a rare crash.  **P3** latent,
cosmetic or off the main path.

---

## Baseline

| Check | Result |
|---|---|
| Mixer test suite (full, including `slow`) | **1 failed**, 1,592 passed, 1 skipped (388 s) |
| bandit (medium and above) | nothing |
| ruff bug rules | no undefined names; five `global` statements |
| Version sync | `__version__`, README header and newest changelog line all `0.222.3` |
| Required docs | `operations.md`, `configuration.md`, `integration.md`, `troubleshooting.md` present |

**Failing test:**
`tests/test_ui_mouse.py::test_performance_tab_audio_process_python_choice_persists`.
It picks the last option of the "Audio process Python" row and asserts it is
APP PYTHON.  `ui.py:11559-11571` looks for a free-threaded interpreter under
the real `~/.local/share`, and this machine has one, so FREE-THREADED is the
last option.  With `XDG_DATA_HOME` pointed at an empty directory the test
passes.  It is a non-hermetic test, not a product defect.  It is not marked
`slow`, so it is in the drop-in's own pre-commit subset.  Not touched.

---

## Summary

| ID | Sev | Part | One line |
|---|---|---|---|
| P1-1 | P1 | engine | The phaser is a per-sample Python loop: on both FX units every block misses its deadline — **VERIFIED** |
| P1-2 | P1 | engine | Naming an output device makes `start()` raise and leaves the mixer silent — **VERIFIED** |
| P1-3 | P1 | autoplay | A headless run reports "over" about 25 s into its last track — **VERIFIED** |
| P1-4 | P1 | autoplay | An empty or missing set never reports its end; an unattended run sits silent — **VERIFIED** |
| P1-5 | P1 | library | The library audit classes playable tracks as "broken" and quarantine moves them — **VERIFIED** |
| P1-6 | P1 | UI | A click on empty space in SETTINGS grabs the deck fader hidden behind it — **VERIFIED** |
| P1-7 | P1 | UI | One double-click on "Delete file…" deletes the file, permanently on this install — **VERIFIED** |
| P1-8 | P1 | UI | Nothing stops a load or eject on a playing deck — **VERIFIED** |
| E-P2-1…16 | P2 | engine | 16 findings; see the engine table |
| C-P2-1…13 | P2 | control | 13 findings; see the control table |
| U-P2-1…11 | P2 | UI | 11 findings; see the UI table |
| P3 | P3 | all | 31 findings; see the P3 tables |

---

## P1

### P1-1. The phaser breaks real time — VERIFIED

`dsp.py:667-675`.  `Phaser.process` runs four allpass stages as a Python loop
over every sample of the block.

Block-render timing, four decks, block 512 at 48 kHz, budget 10.67 ms:

| Configuration | mean | p99 | max | blocks over budget |
|---|---|---|---|---|
| Phaser on both FX units | 15.5 ms | 20.4 ms | 43.9 ms | 1,500 of 1,500 |
| Phaser on one unit | 7.8 ms | 13.8 ms | 14.2 ms | 17 of 800 |
| Everything else on (split stems, keylock, EQ, filter, ×4) | 3.1 ms | 3.6 ms | 7.0 ms | 0 |

Engaging the phaser on both units underruns the output continuously.  On one
unit it uses about 73 % of the budget on its own.  Nothing else in the engine
comes close: the next heaviest configuration is 29 %.  `TranceGate`
(`dsp.py:631-633`) has the same per-sample shape but stays inside the budget.

**Fix:** vectorize the cascade (`lfilter` with carried state, as the EQ does).

### P1-2. A named output device leaves the mixer silent — VERIFIED

`mixer_engine.py:1768` (use), `1837` (the only place the attribute is
created), `1829` / `1850` (callers).  `_owned_device_names` is created only
inside the handler for a failed 4-channel open, so `_note_owned_device()`
raises `AttributeError` the first time `start()` runs with a non-empty
`device`.

- **Stereo device named in `output_device`:** `start()` raises after the
  stream is opened; no writer thread starts; a later `start()` returns early
  because a stream object exists.  No audio.
- **4-output device with `headphone_device = "same"`:** the 4-channel stream
  opens, the `AttributeError` is caught as "4-channel open failed", that
  stream is dropped without `close()`, and a second, stereo stream is opened
  on the same hardware with no cue bus.

The owner's current config has `output_device = ""`, which does not reach
this path.  Anyone who names a device does.

Also by reading: the separate headphone stream (1851-1878) is never recorded
as owned, and `close()` never clears the set.

**Fix:** create the set in `__init__`; record the headphone device; clear on
`close()`; close the 4-channel stream in the except branch.

### P1-3. "Night over" fires when the last track starts, not when it ends — VERIFIED

`autoplay.py:1541-1557`, `dj_mixer_controller.py:2063-2071`.  About 20 s into
the last track, `_arm_next` finds nothing left to arm and latches
`_night_over`.  The controller then publishes
`{'phase': 'over', 'seconds_left': 0.0}`, overriding `final`.

Reproduced with a fake engine and clock: 25 s into the last track, the deck
is playing with 35 s remaining and the payload already says `over`.

In a headless `--dj-mixer-source` run, Auto VJ's finale fires on that signal
and the app exits, cutting the last track from the run and from the
recording.  The exit chain through Auto VJ (`auto_vj.py:9920-9960`,
`unicornviz/__main__.py:280`) is by reading; Auto VJ was not run.

**Fix:** latch only when the list is exhausted and no primary deck is
playing; keep `final` until then.

### P1-4. A set that cannot play never ends — VERIFIED

`autoplay.py:1525-1527`, `1017-1035`; `dj_mixer_controller.py:938-941`,
`989-996`.  With an empty set, `_arm_next` does nothing and
`playlist_progress()` returns `None`.  With `autoplay_boot_set` naming a set
that does not exist, boot autoplay returns without arming (by reading).  No
`over` is ever published, so an unattended run sits silent until it is
killed.

Reproduced for the empty set: 12 s of ticks, nothing fired, progress `None`.

Related (C-P2-1): a set on an unmounted drive flags every row "no auto" and
then idles the same way.

**Fix:** for boot autoplay, treat "nothing armable" as terminal and publish
`over`, or exit with an error.

### P1-5. The library audit quarantines playable tracks — VERIFIED

`browser.py:1073-1076`.  A track is classed "Broken / undecodable" when it
has any store record but no `duration_s`.  A track that was loaded on a deck
and never batch-analyzed has exactly that record: `deck.marks()`
(`deck.py:1466-1483`) carries no duration.  So does anything analyzed before
0.71.0.

LIBRARY → QUARANTINE (two presses) then moves those files to
`_quarantine/broken`.  Their set rows go dead, and PRUNE THEM deletes the
rows.

Reproduced on scratch files: a loaded-only `alpha.flac` was reported broken,
moved, left a ghost in the set "Friday", and was pruned from it.

Nothing is deleted.  UNDO works for the last sweep of the session only (the
record is in memory); after that, recovery is by hand from the manifest.

**Fix:** call a file broken only when it is missing or a decode actually
failed, never from a missing duration.

### P1-6. A click in SETTINGS dead space moves a hidden deck control — VERIFIED

`ui.py:4630-4634`, `2154-2175`.  A left click on the settings panel's
background calls `_begin_drag`, which takes the topmost hit region with no
check that it belongs to the open panel.  Absolute faders jump to the click
position on press.

At 1280×800 with SETTINGS open: a click on blank space near the panel's left
edge set deck A's key shift to +5.3 semitones; another set its tempo to
−7.1 %; the same on deck B.  490 sampled dead-space points sit over a hidden
control (KEY, TEMPO, channel and master faders, EQ, filter, trim, FX).

Wheel and right-click already filter by `_modal_hit_ids`; left-click does
not.

**Fix:** in that branch, only begin a drag on hits owned by the open panel.

### P1-7. A double-click deletes a file — VERIFIED

`ui.py:3454-3461`, `12716-12717`, `3737-3776`.  "Delete file…" arms on the
first press and confirms on the second, on the same row, with the label
swapped in place.  A double-click satisfies both.

What "delete" does: `send2trash` is not in either requirements file and is
not installed in this environment, so the delete is a permanent `unlink()`.
A trash failure also falls through to `unlink`.  The music-folder fence is in
place (regular file, resolved under the resolved root).

**Fix:** put the confirm in a different position; ignore presses with
`clicks >= 2` or within about 400 ms of arming; add `send2trash` to the
drop-in requirements and refuse, not unlink, when it is missing.

### P1-8. No protection for a playing deck — VERIFIED

`ui.py:7103-7104`, `2621-2623`, `5289-5325`, `5357-5360`, `3373-3374`;
`browser.py:596-606`; `deck.py:902`.

- **EJECT** is enabled whenever a deck is loaded.  One click on a playing
  deck stops and unloads it.
- **Double-clicking a library row** while decks 1 and 2 both play loads onto
  the quieter of the two playing decks.
- **Dropping a row on a playing deck** loads it.
- **"Load → Deck N"** in the row menu has no check (by reading).

Playback stops at swap-in.

**Fix:** refuse load and eject on a playing deck in the UI (a toast, or a
hold / second gesture), and never choose a playing deck as the "opposite".

---

## P2 — engine

| ID | Where | What | Evidence | Fix |
|---|---|---|---|---|
| E-P2-1 | `deck.py:648-652`, `2877`; `dsp.py:128-133`; `mixer_engine.py:1348-1354` | One NaN or Inf sample in a track turns the master to NaN from that block on, even with that deck's fader at 0.  EQ state stays NaN after a clean track is loaded; `master_level` stays NaN until restart | **VERIFIED** | `nan_to_num` at decode; a finite guard on the mix |
| E-P2-2 | `dsp.py:563-595`, `deck.py:2874-2875` | Keylock (on by default, active on every synced deck) delays audio 40-50 ms and splits each transient into two copies about 44 ms apart.  SYNC phase-locks on cursors, so synced beats flam.  Re-engaging plays about 43 ms of stale audio, possibly from the previous track | **VERIFIED**: click at +1,896 / +2,310 samples with two copies; +4 samples and one copy without keylock | Keep feeding the ring in bypass; crossfade on engage; report the latency |
| E-P2-3 | `mixer_engine.py:1338-1348`; `dsp.py:109-113` | Channel fader, crossfader, master and trim are applied as one scalar per block with no ramp; the filter swaps coefficients in one step | **VERIFIED**: boundary jump 0.41 against a normal sample step of 0.0065 (×63) for a fader cut on a 100 Hz sine.  EQ kill and a 127-step filter sweep are clean | Per-block linear gain ramps |
| E-P2-4 | `dsp.py:519-529`, `416-446` | Flanger and reverb delay lines read before writing, which is only valid when the delay is at least one block.  The flanger (24-288 samples) becomes a 20 ms slapback; the reverb's diffusion stage is not the documented 1.7 / 5 ms | **VERIFIED**: flanger impulse taps at 100 and 1,216 (intended delay 156); output differs by block size | Sub-block processing |
| E-P2-5 | `deck.py:147-158`, `668-672` | Every file libsndfile opens (WAV, FLAC, OGG, AIFF, MP3) is resampled 44.1 → 48 kHz by linear interpolation | **VERIFIED**: 10 kHz −1.5 dB with an image at −21 dBc; 15 kHz −3.4 dB / −11.5 dBc; 18 kHz −5.1 dB / −6.5 dBc.  Pitch and length are correct | `resample_poly`, already used in `stems.py` |
| E-P2-6 | `mixer_engine.py:1635-1658` | `write()` returns "underflowed" as a value; the loop discards it.  Device underruns are never counted or logged | **VERIFIED**: 129 underflows returned, `underflow_count = 0` | `if stream.write(block): count` |
| E-P2-7 | `mixer_engine.py:1653-1658` | A `PortAudioError` from `write()` is treated as "block consumed" with no sleep: the writer spins and the decks race forward (72× real time in the test).  Which device events raise was not confirmed | **VERIFIED** (loop) | Count consecutive errors; sleep; reopen |
| E-P2-8 | `mixer_engine.py:1637-1652` | A separate headphone device is fed by a second blocking write in the same loop.  A 250 ms stall of the phones device stalls the master for 262 ms | **VERIFIED** | Its own writer thread |
| E-P2-9 | `deck.py:878-880`, `1481`, `2231-2243`; `analysis_job.py:65` | A substituted 120.0 BPM (silence, sub-1-second material) is stored as an ordinary `bpm` and comes back as measured on the next load, so quantize runs on a fabricated grid | **VERIFIED**: first load `measured False`, reload `measured True` | Store a measured flag |
| E-P2-10 | `structure.py:547-548`, `583`, `606` | `beat_offset` is treated as where the music starts, but it is a downbeat anchor the DJ can place anywhere.  Everything before it is dropped from structure analysis | **VERIFIED**: anchor at bar 70 labels the quiet intro "drop" | `off % bar_len` |
| E-P2-11 | `analysis_job.py:30`, `64`, `84` | Analysis hashes at a hard-coded 48 kHz; the deck hashes at the configured rate.  With `samplerate = 44100`, sections and `analysis_version` are orphaned and the track reads as un-analyzed after every load | By reading; hash difference **VERIFIED** | Pass the engine rate |
| E-P2-12 | `deck.py:2615-2619` vs `2819-2825` | The transform gate is not applied to the FX send, so with a stem mask the FX keep receiving audio through the closed gate | **VERIFIED**: send RMS 0.14 with the gate closed | Gate the send |
| E-P2-13 | `deck.py:980`, `1168`; `sampler.py:78`, `100` | On Python 3.11 / 3.12 every deck load, stem attach and pad load raises (`SharedMemory(track=)`), after `_swap_in` has already reset the deck: new track name, old audio, old BPM, no cues | **VERIFIED** by injection | Gate the argument in core; compute before mutating |
| E-P2-14 | `mixer_engine.py:1586-1595`, `1417-1418` | A write error ends the recording with one log line and no operator signal; the rest of the set is not recorded | **VERIFIED**.  Normal recordings and a kill mid-recording both left complete, readable files | A `recording_error` state for the UI |
| E-P2-15 | `deck.py:154-158`, `636-691` | Whole-file temporaries on load: the linear resampler peaks at 6.5× the track, auto-gain at 3×.  A 60-90 minute mix would need roughly 9-13 GB transiently (extrapolated, not run) | **VERIFIED** multipliers on a 5-minute track | Chunk both |
| E-P2-16 | `mixer_engine.py:1348`, `912` | The master "soft-clip guard" is an always-on `tanh`: 0.5 in gives 0.462 out (−0.68 dB, third harmonic −34 dBc); full scale gives −2.4 dB.  Recordings are post-`tanh`; the phones bus is shaped twice | **VERIFIED**.  A single deck's render is bit-exact | A limiter that is transparent below a threshold |

**Timing, everything except the phaser** (mean ms per 10.67 ms block, four
decks): flat 0.42; +EQ +filter 0.68; keylock 1.10; split stems ×4 1.45;
reverb 0.94; 16 sampler voices 0.44.  Nothing scales with track length, no
lock is taken on the render path, and a 3.5 s track load in the same process
did not stall it.

**Signal checks that passed:** deck render bit-exact; crossfader curves and
hamster inversion; loop boundaries within 0.011 samples; 6,000 fuzzed
cue/seek/scratch/jump operations with no exception and no non-finite output;
end of track; four decks at full scale stay at or below 1.0; mono and
44.1 kHz files play at the right pitch and length; the cue bus is independent
of the master; BPM, key, structure and hash are deterministic.

**`ANALYSIS_VERSION`:** the discipline holds.  An AST comparison of the
analysis functions at the version-3 commit (`876cdb0`) against HEAD found no
change to stored results without a bump.  Every reader treats a missing or
older version as stale, the stale count is surfaced, and re-analysis is
operator-triggered.  One gap: `browser.py:1546-1547` writes `sections` only
when the new result is non-empty, so a re-analysis whose structure pass fails
keeps the old strip under a current version stamp.

---

## P2 — control, library, hardware

| ID | Where | What | Evidence | Fix |
|---|---|---|---|---|
| C-P2-1 | `autoplay.py:1434-1437`, `1507-1515`; `browser.py:1286-1302` | Three paths permanently flag good tracks "no auto", and nothing clears the flag: ejecting the armed deck; a set on an unreachable drive (6 of 6 rows flagged, then idle); any exception from an analysis job, not only a decode failure | **VERIFIED**, all three | Flag only on a real decode failure of an existing file; clear on the next success |
| C-P2-2 | `track_store.py:452-470`, `sets.py:345-362`, `state.py:295-312` | A truncated or wrong-shape store loads as empty and the next save overwrites it.  No quarantine copy.  Core's `safe_store` is not used.  The two rolling backups survive, but the session runs on an empty store, and once re-analysis passes half the backup's size the backups rotate out | **VERIFIED** | Rename the bad file aside and load `.bak1`, or adopt `safe_store` |
| C-P2-3 | `track_store.py:578-590`, `549-558` | A failed write is never retried: the dirty flag is cleared before the write, so the next `save()` and the shutdown `flush()` write nothing | **VERIFIED**: cue not on disk after the directory became writable again | Re-set the flag on failure |
| C-P2-4 | `dj_mixer_controller.py:1403-1415`, `1349-1381` | The periodic deck flush writes the deck's BPM, key and grid back over a re-analysis of a loaded track, while `analysis_version` still reads current | **VERIFIED** as a store-call sequence | Push the new values into the deck, or stop flushing analysis-owned fields |
| C-P2-5 | `tags.py:111-126`, `track_store.py:219-222` | A tag edit changes mtime, the path link misses, and the track reads as un-analyzed until re-decoded.  The toast says cues and loops are kept.  The tag write is in place with no backup and no music-folder fence | **VERIFIED**: `bpm 128.0 → None` | Re-link the path after the write |
| C-P2-6 | `autoplay.py:690-702` | Starting the armed deck by hand does not consume the playlist cursor: the same track is armed on the other deck and plays twice | **VERIFIED** | Consume in `_reacquire` |
| C-P2-7 | `autoplay.py:388-415`, `519-528`, `1841-1844` | A mode change mid-crossfade leaves both decks playing with the fader frozen at 0.5.  By reading: turning autoplay off during a stem-blend hold leaves vocals and melody muted; ejecting the held deck stalls `tick()` | **VERIFIED** (crossfade case) | Finish or roll back on a mode change |
| C-P2-8 | `midi_map.py:124`, `rev1_input.py:557` | Generic mapping files cannot drive any deck fader, EQ or pitch: `PLATTER_CC` is an `int` and the decoder tests `cc in` it.  The documented `eject` note has no handler | **VERIFIED** with the shipped example mapping: `TypeError` per message | Make it a `frozenset` |
| C-P2-9 | `dj_mixer_controller.py:1937-1947` | `_exchange_bpm` looks only at decks A and B: with only C or D playing nothing is published; with both playing it always publishes A's track | By reading | Use `live_deck_key` |
| C-P2-10 | `library.py:667-670` | Sorting by artist or genre compares the first character only | **VERIFIED**: `Bob Zed` before `Bob Alpha` | Use the whole string |
| C-P2-11 | `dj_mixer_controller.py:1433-1434`; `rev1_leds.py:199-254`; `s4mk3_hid.py:198-204` | Controller hot-plug is still not handled: "open" is set on "not found yet"; the LED port is opened once; the S4 reader exits on the first read error | **VERIFIED** (LED side: 200 failed sends, no reopen) | Retry from `update()` |
| C-P2-12 | `unicornviz/app.py:7355-7363` (core) | Autoplay advances only inside the subsystem update, which core skips while the ProjectM manager is open: a transition due then does not fire | By reading | Tick autoplay regardless |
| C-P2-13 | `dj_mixer_controller.py:1139-1145`; `mixer_engine.py:1477-1479`, `1660` | Split-by-track recording blocks the caller for 32-1,014 ms at each track change; with the audio helper the caller is its single command thread, so every deck command queues behind it.  A render error logs a traceback ten times a second (09-30 P3-11, open) | **VERIFIED** (the wait) | Hand the split to the flusher; throttle |

**Destructive operations** (full table in the reader's notes): single delete
(P1-7); quarantine and consolidate (two-press, fenced to the music folder,
never overwrites, writes a manifest); undo (last sweep only); tag save (in
place, unfenced); prune (store only); backup restore (two-press).  A
cross-filesystem move that fails leaves the source intact.

**Stores:** all three (`dj_mixer_tracks.json`, `dj_mixer_sets.json`,
`dj_mixer_state.json`) are written atomically (temp file in the same
directory, then `os.replace`).  The track store keys marks by audio hash and
links paths by absolute path, size and mtime, so marks survive a moved
library but every set, favorite and path link goes dead until each file is
decoded again.  `schema_version` is written and never checked.  Hash records
are never pruned.  Nothing writes `config.toml`.

---

## P2 — UI

| ID | Where | What | Evidence | Fix |
|---|---|---|---|---|
| U-P2-1 | `ui.py:1893-1902`, `1526-1542` | Key auto-repeat defeats the Esc-Esc close guard: holding Esc closes the window, stops the engine if KEEP PLAYING is off, and in hosted mode exits the app | **VERIFIED** | Ignore repeat key-downs |
| U-P2-2 | `ui.py:5002-5008`, `4996-5001`, `2286-2316`, `4744-4752`, `4543-4582`, `10550-10581` | Every "press twice" guard is satisfied by one double-click: ANLZ ALL, RE-ANLZ, grid reset ("no undo"), the title-bar X (whole-app exit); by reading also backup RESTORE, PRUNE and QUARANTINE | **VERIFIED** for the first four | One shared arm helper that rejects double-clicks |
| U-P2-3 | `ui.py:2656-2658`, `4978-4980`, `9147-9182` | THROW OUT cannot be called off: right-click cancel leaves the fade running and the dry signal killed; an auto-fade arrow mid-run does not cancel, and the deck is ejected anyway when the fade returns | **VERIFIED** | One cancel routine; eject only if the fader ended on the far side |
| U-P2-4 | `ui.py:1831-1833`, `1779` | Below 960×600 every click is hit-tested at the wrong place (the tracked size is clamped, the mouse scale is computed from the clamp).  The window has no minimum size | **VERIFIED** at 800×480: gear drawn at (601, 34), hit-tested at (721, 42) | Drop the clamp or set an SDL minimum |
| U-P2-5 | `ui.py:2644`, `2662`, `6291-6293` | Right-clicking the FX D/W dial resets it to 50 % instead of opening the stem popover, which is therefore unreachable by mouse | **VERIFIED** | Test the popover first |
| U-P2-6 | `ui.py:4875-4881`, `4671-4674`, `1887-1892` | A focused folder box keeps the whole keyboard after SETTINGS closes; the next Enter re-applies the music folder and rescans mid-set | **VERIFIED** | Clear focus on close |
| U-P2-7 | `ui.py:5284-5286`, `5333-5360`, `5314-5322` | Drag-and-drop and double-click load by a row index captured at press; if the view re-orders first, a different track is loaded | **VERIFIED** | Capture the path |
| U-P2-8 | `ui.py:11189`; `track_store.py:111-123` | After an audit has run, the LIBRARY tab parses every store backup from disk on each frame: 290 ms a frame against 14 ms, on the raster thread | **VERIFIED** on a 6.9 MB store | Cache by mtime and size |
| U-P2-9 | `ui.py:2811-2812`, `3386-3388`, `3466-3502` | "Remove all cues" / "Remove all loops" act on one click with no confirm and no undo, in the store and on any deck holding the track | By reading | Arm and confirm |
| U-P2-10 | `ui.py:2629-2631`, `1938-1958`, `5682`, `10726-10742` | Modals do not own the pointer: with the tag editor open, right-click and wheel outside it still act (a hot cue was cleared, a gain moved); with the rename prompt open, left clicks drive the board and a double-click inside the prompt loads the row beneath | **VERIFIED** | One "modal open" gate before hit-testing |
| U-P2-11 | `ui.py:13222-13227`, `8321-8323`, `6646` | A browser row with BPM `inf` or negative never leaves the octave-fold loop: the raster thread spins for the session.  A deck BPM of NaN or `inf` raises every frame.  Whether the analyzer or store can produce such values was not established | **VERIFIED** (`inf` row) | `isfinite(bpm) and bpm > 0` |

**Frame cost** (four decks, 5,000 rows): the GPU draw-list path takes
10-15 ms a frame at any size up to 4K.  The Pillow path, always used in
hosted mode, takes 17 ms at 1280×800, 29 ms at 1080p and 104 ms at 4K.  The
browser is virtualized: 50,000 rows cost 9.5 ms.  The whole frame is rebuilt
every tick whether or not anything changed.

---

## P3

### Engine

| ID | Where | What |
|---|---|---|
| E-P3-1 | `deck.py:927` vs `2597`; `1328`; `2401-2408` | The render path re-reads `_samples` / `_stems` and writes `_cursor` after a concurrent load or eject reset them: a new track can start parked at the old cursor, and an eject mid-block raises (traceback plus a 100 ms sleep) — **VERIFIED** by interleave.  `_pending_q` is rebuilt on one thread while another appends (by reading) |
| E-P3-2 | `structure.py:393-396`, `557-566`; `analysis_job.py:127` | The "unique loud block" rule tests `vocal <= 0`, which cannot hold (floor 1e-6); `analysis_job` never passes stems, so the vocal and drum rules are unreachable in production — **VERIFIED** |
| E-P3-3 | `unicornviz/remote_objects.py:303-357` (core) | A helper that crashes leaves every track and stem segment in `/dev/shm` (core audit P2-11).  Playback recovers in-process |
| E-P3-4 | `deck.py:1498-1552` | One malformed store value (`hot_cues: ['x']`, `bpm: 'fast'`, NaN) makes that track fail to load every time — **VERIFIED** |
| E-P3-5 | `stems.py:567-579` | The Demucs subprocess has no timeout and no stored handle, and the path is passed without `--` (not run) |
| E-P3-6 | `deck.py:1607-1608`, `2003-2005` | `set_loop` does not clamp to the track length: a loop straddling the end plays 47 % silence and never stops — **VERIFIED**.  Reads stay in bounds |
| E-P3-7 | `bpm.py:70`, `75`, `86-92` | A constant tone, white noise and a track with one NaN all return a "measured" BPM — **VERIFIED**.  BPM cost is quadratic in length: 6.4 s for 40 minutes |

### Control, library, hardware

| ID | Where | What |
|---|---|---|
| C-P3-1 | `s4mk3_hid.py:133-134`, `201`; `docs/hardware-bringup-s4mk3.md:32` | The code uses the `hid` package API; the doc and the log say `pip install hidapi`, whose API differs (SUSPECTED; neither is installed) |
| C-P3-2 | `s4mk3_input.py:549-557`; `rev1_input.py:304-305` | Wheel touch is released on whichever deck is selected at release time, so a touch held across a deck switch leaves the old deck in scratch-hold — **VERIFIED** for the S4 |
| C-P3-3 | `rev1_input.py:577-605` | An LSB arriving before any MSB is combined with MSB 0: the crossfader jumps from 0.5 to 0.0055 — **VERIFIED**.  A new MSB is still paired with the old LSB (09-30 H9, open) |
| C-P3-4 | `rev1_input.py:117`, `286-288` | SHIFT state survives a mixer close and reopen — **VERIFIED** |
| C-P3-5 | `track_store.py:139-159` | Backups are made with `copy2`, so the 15-minute spacing check passes twice at session start and the older backup is evicted early — **VERIFIED** |
| C-P3-6 | `sets.py:354-359`; `track_store.py:230` | A wrong-typed `favorites` aborts the sets load partway and the loss is saved; a non-dict track record raises on the render thread — **VERIFIED** |
| C-P3-7 | `dj_mixer_controller.py:1837-1866`, `598` | Four per-frame calls are outside any guard: a raise skips the rest of that frame's mixer update; a failed boot `set_mode` is not retried |
| C-P3-8 | `autoplay.py:1237-1260`, `1521-1522` | After the list ends, every tick copies the whole list once per row; a track shorter than about 20 s ends before its successor is loaded |
| C-P3-9 | `native_dialog.py:71-74`, `102` | kdialog gets the initial path as a positional argument (a leading `-` parses as an option); the PowerShell call has no `CREATE_NO_WINDOW` |
| C-P3-10 | `xdjxz_map.py:20`, `47-49` | The docstring says decks are on channels 2-5; the table uses 0-3.  If the prose is right, every deck is off by one (SUSPECTED; no source document) |
| C-P3-11 | `s4mk3_input.py:573-596` | Every analog report rewrites the crossfader and all four channel strips, so ADC jitter would fight autoplay (device unvalidated) |
| C-P3-12 | `browser.py:667-671` | `sets.note_played()` saves from the loader thread while the UI thread mutates and saves the same unlocked store |
| C-P3-13 | `rev1_leds.py:268-274`; `rev1_input.py:672-702`, `157-161` | 09-30 hardware items still open: beat flash ignores the downbeat offset; only pitch has soft takeover; `rev1_log_notes` logs note data at INFO |

### UI

| ID | Where | What |
|---|---|---|
| U-P3-1 | `ui.py:1917-1919`, `3790-3809` | Enter auto-repeat loads the selection three times; a loaded, paused deck counts as "idle" and is replaced — **VERIFIED** |
| U-P3-2 | `ui.py:5111-5116`, `5267-5268`, `1997-1999` | Search and EDIT TAGS build text from keysyms: no shifted punctuation, no non-ASCII, Ctrl+V types "v" (09-30 U16, open) |
| U-P3-3 | `ui.py:11741-11772`, `11288`, `10033` | Open panels do device and store work every frame: two `query_devices()` and a MIDI port scan (GENERAL), a stale-analysis scan at 36 ms (ANALYZER), an unanalyzed count at 31 ms (SMART LIST) — **VERIFIED** |
| U-P3-4 | `ui.py:1974` | A horizontal wheel event moves the browser selection down four rows — **VERIFIED** |
| U-P3-5 | `ui.py:2097-2111`, `1814-1844` | Held controls (scratch touch, FX slot, ROLL pad) have no release path except a left button-up; they stay engaged after focus loss in the harness (SUSPECTED: SDL may synthesize the button-up) |
| U-P3-6 | `ui.py:13072`, `7940`, `1335` | On the Pillow path a window 360 px tall or less raises on every frame — **VERIFIED** |
| U-P3-7 | `ui.py:1608-1662`, `11924-11927` | `_ui_render_ms` grows without bound when the diagnostic interval is "off" |
| U-P3-8 | `ui.py:5401-5402` | Crossfader auto-fade and THROW OUT advance only while the UI renders: paused or hidden, the fade stops — **VERIFIED** |
| U-P3-9 | `ui.py:8099-8103`, `8064`, `9036`, `13102-13104` | Wrong readouts: the stacked view shows base BPM, not pitched tempo, and labels lanes C/D; the ►► tooltip says "deck 2" under hamster; the "► SET" marker compares a truncated name |
| U-P3-10 | `ui.py:1868-1874`, `10724`; `5650`, `10742` | Length caps truncate silently (48 characters in rename, 240 on paths); `_modal_rects` is rebuilt mid-frame while the main thread reads it (SUSPECTED race) |
| U-P3-11 | `ui.py:648-664`, `667-746`, `8380-8400`, `6950-6963`, `6142-6146`, `6996-7007`, `9120-9123`, `12699-12703`, `12453-12477`, `13158`, `11585-11607` | 09-30 UI items still open: U13 glyph cache keyed by `id(font)`, U14 unscaled polygon/arc, U15 tooltip at `ui_scale` ≠ 1, U17 beat pulse ignores the offset, U18 sparkle sprites, U19 pad-tile cache never evicted, U20 THROW OUT on decks 3/4, U21 context menu not clamped, U22 wrong tooltip, U23 stale `config.toml` advice, W4 device picker stores bare names |

---

## Status of earlier findings

| Earlier ID | Status |
|---|---|
| U1 hosted mode drops all input | Fixed (`ui.py:1808-1813`) |
| U3 quantized ROLL / TRANS tap sticks | Fixed, mouse and REV1 paths |
| U4 THROW OUT under hamster | Fixed; new problems in U-P2-3 |
| U5 Smart List same-minute merge | Fixed in the UI; `sets.add` itself still appends to an existing name |
| U6 backup restore undone by the next save | Fixed, re-verified on scratch files |
| U7 display 0 unreachable | Fixed |
| U13-U23, W4, W17 | Open (U-P3-2, U-P3-11) |
| H13 REV1 LEDs dead | Fixed (`dj_mixer_controller.py:1313-1326`); a vestigial hook remains at `mixer_engine.py:1176` |
| H1 no reconnect after unplug | Open (C-P2-11); core's retry is one-shot (core audit P2-10) |
| H7, H8, H9, H12 | Open (C-P3-3, C-P3-13) |
| W7 console window for the folder picker | Open (C-P3-9) |
| 09-30 P3-11 split-recording busy-wait | Open (C-P2-13) |
| 2026-07-11 audio degradation | The original mechanism cannot recur: there is no Python audio callback and the engine runs in its own process by default.  Residual exposure: a failed attach silently keeps the engine in-process; E-P2-6 hides underruns; P1-1 reproduces the symptom |

---

## Rule violations (CLAUDE.md)

- **Hotkeys documented outside `HELP_ENTRIES`.**  `HELP_ENTRIES`
  (`dj_mixer_controller.py:193-199`) lists five keys.  `ui.py` hard-codes key
  hints into on-screen text at 15 sites, including a full "HOTKEYS &
  GESTURES" table (`ui.py:9517`).  Handled but unlisted: the tag-editor and
  prompt keys, Ctrl/Cmd+V, and Ctrl+click / Shift+click gestures.
- **`config.toml` advice.**  Live UI text still tells the owner to edit
  `config.toml` (`ui.py:11399-11401`, `12112-12116`, `13158`), against the
  move to the config menu.
- **MIDI note data at INFO** when `rev1_log_notes` is on
  (`rev1_input.py:158`, `rev1000_input.py:167`).
- **`global`:** `dj_mixer_controller.py:81`, `100`; `mixer_engine.py:383`;
  `s4mk3_hid.py:46`.
- **`# noqa: SLF001` in drop-in code:** eight sites
  (`dj_mixer_controller.py:1342-1343`, `1565`; `analysis_job.py:64`, `86`,
  `99-100`, `136`).
- **Public Runtime Surface:** no `app._private` access anywhere; everything
  goes through `vj_api`.  Clean.
- **Config keys:** 14 keys are read but absent from `docs/configuration.md`
  (`keep_playing_on_close`, `persist_state`, `rev1_log_notes`, the three
  store paths, `spin_transitions`, `start_two_deck` and others); three are
  validated but never read (`master_gain`, `samples_dir`, `quarantine_dir`);
  about 25 are read with no validation, several through bare `float()` /
  `int()`.
- **Optional dependency:** `send2trash` is guarded but in no requirements
  file (P1-7).
- No `print()`.  No `eval`, `pickle`, or `shell=True`; every subprocess call
  uses a list.

---

## Design debt (one line each)

1. `rev1000_input.py` and `rev1000_leds.py` are near-verbatim copies of the
   REV1 files (1,173 lines), so every REV1 defect exists twice.
2. Confirmation is implemented six different ways in `ui.py`; none is
   double-click-safe (U-P2-2).  Text entry is implemented three ways.
3. Hit regions are plain tuples matched by string in a 600-line `if/elif`
   (`ui.py:4475-5080`); modal ownership is tracked by `id()` of per-frame
   objects (the root of P1-6 and U-P2-10).
4. Audio-affecting automation (auto-fade, THROW OUT eject) lives in the draw
   path (U-P3-8).
5. Settings panels call the audio device, MIDI and disk from the raster
   thread.
6. A render exception is logged without a traceback (`ui.py:1649`).
7. `sets.py:35-39` re-executes `track_store.py` on every sets save to reach
   one function.
8. "Temporary instrumentation (2026-09-18)" still runs in
   `browser.snapshot()` every frame (`browser.py:1584-1592`).
9. A headless run that shares `runtime/` saves its autoplay board over the
   owner's session state at shutdown (`dj_mixer_controller.py:859-867`,
   `1604`).
10. The writer thread makes about 40 small allocations per block and logs
    from the real-time path.
11. `operations.md` and `integration.md` are dated 0.214.11 and 0.194.1.

---

## What this audit did not do

- No audio device, MIDI port or HID device was opened; no real helper
  process, no Demucs run, no Windows test, no real SDL window or GL context.
- No real music was decoded or analyzed, and no detection accuracy was
  judged.  A 3-hour file was extrapolated from 5-40 minute runs.
- The drop-in's prose docs were grepped, not read.  Its tests were read only
  where needed to build harnesses.
- Did not check whether the analyzer or store can actually produce the
  non-finite BPM values of U-P2-11, or which device events trigger E-P2-7.
- Did not fix anything, including the failing test.  P1-2, P1-6, U-P2-1 and
  C-P2-3 are each a change of a few lines.
