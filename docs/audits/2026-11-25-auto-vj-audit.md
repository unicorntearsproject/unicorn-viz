# Auto VJ drop-in audit — `drop-ins/auto-vj-01` (2026-11-25)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-25

Scope: the whole `auto-vj-01` drop-in at submodule commit `a5b11e5`
(drop-in `1.0.0-rc.155`, detector rc.46, director rc.22, recommender rc.50):
`auto_vj.py` (9,982 lines), `beat_grid.py` (8,073), `fire_dj_celebration.py`
(1,077), the seven scripts in `tools/`, `install.sh`, and the sync obligations
in CLAUDE.md (weights doc, subsystem versions, training-kit payload dicts,
README).

**Tuning was out of scope.**  No weight, threshold, window or cooldown value
is judged here.  Findings are code that does not do what it says.

Method:

- `auto_vj.py`: lines 1-1499 read in full; from 1500 on, every code line read
  in order, with long comment runs cut to their first three lines (1,141
  comment lines not read).
- `beat_grid.py`: lines 1-4442 and 7342-8073 read in full.
  `_BeatTrackerV3Base` (4443-7341) was proven byte-identical to the v2 class
  by an `ast` diff except for `_estimate_tempo_acf`, whose differing hunks
  were read.
- `fire_dj_celebration.py`, `install.sh`: read in full.  Tools: run with
  `--help` and with defaults.
- Trackers and a bare controller were driven with synthetic input (click
  trains, silence, NaN, clock jumps, 24-144 fps).  The celebration overlay was
  rendered in an offscreen GL context.  The app was not launched and no real
  music was used.
- **VERIFIED** = reproduced by script; the four most severe were re-run by a
  second reader.  **SUSPECTED** marks what could not be confirmed.

Repro scripts: `/var/tmp/uv-dropin-audit/{autovj,beatgrid,mine}/`.

Baseline: the Auto VJ tests live in the core suite (3,240 passed, 4 skipped at
this commit).  They miss every finding below.

`auto_vj.py` has not changed since the 2026-09-30 audit (same commit, same
line numbers), so **all 18 findings of
[2026-09-30-auto-vj-director-recommender-audit.md](2026-09-30-auto-vj-director-recommender-audit.md)
are still open**.  They are listed in the status table, not repeated in full.

Severity: **P1** can freeze the show or leave part of Auto VJ dead for the
session; **P2** wrong behavior an operator notices, or training data silently
wrong; **P3** latent, cosmetic or off the live path.

---

## Summary

| ID | Sev | Subsystem | One line |
|---|---|---|---|
| P1-1 | P1 | recommender | The decider cannot leave an audio profile the BPM prefilter excludes (09-30 AV-1, open) |
| P1-2 | P1 | director | Toggling off during a pending mode snap blocks all mode changes for the session — **VERIFIED** (09-30 AV-2, open) |
| P1-3 | P1 | detector | A jump in the audio clock stalls the render thread: 1.7 s per day of jump — **VERIFIED** (trigger unconfirmed) |
| P1-4 | P1 | detector | One NaN or Inf audio value kills the drop signals for the rest of the session — **VERIFIED** |
| P2-1 | P2 | config | About 30 `[auto_vj]` keys can never be set from config, even in USER mood — **VERIFIED** |
| P2-2 | P2 | director | A track change swaps the effect during USER hold, PAUSE and the startup grace — **VERIFIED** |
| P2-3 | P2 | director | Deferred drops and mode entries fire ahead of every guard, after an unbounded wait (09-30 AV-3, wider) |
| P2-4 | P2 | training data | Fizzle and blocked-mode counters count frames, not events — **VERIFIED** |
| P2-5 | P2 | detector | One bad onset-flux value blinds the dense envelope for 13-27 s — **VERIFIED** |
| P2-6 | P2 | detector | A render stall over 0.25 s leaves the beat grid offset for about a minute — **VERIFIED** |
| P2-7 | P2 | detector | The silence reset does not reset the v3 posterior; the next track opens as a confident lock on the old tempo — **VERIFIED** |
| P2-8 | P2 | detector | `top_candidates` can stay on the previous track while v3 has moved on — **VERIFIED** |
| P2-9 | P2 | detector | Cadences are counted in frames: detector timing changes with frame rate — **VERIFIED** |
| P2-10 | P2 | celebration | The Fire DJ overlay never reaches the screen — **VERIFIED** |
| P2-11…16 | P2 | director | 09-30 AV-4 to AV-9, unchanged |
| P3-1…P3-21 | P3 | various | See the P3 table |

---

## P1

### P1-1. The decider cannot leave a prefiltered-out profile (09-30 AV-1)

`auto_vj.py:7452-7484`, `7667-7680`.  When the active audio profile has been
excluded by the BPM prefilter, the switch gate still compares the winner
against the runner-up (or against itself) instead of against zero, so the
switch never clears its margin.  Code is byte-identical to what 09-30
reproduced (a 174 BPM track never gets the drum-and-bass profile).

**Fix:** when the active key was prefiltered out, gate with `current_prob = 0`.

### P1-2. Toggle-off during a pending mode snap — VERIFIED (09-30 AV-2)

`auto_vj.py:5286-5326`, `8337`.  `toggle()` clears the grid's callbacks and
`_drop_pending` but not `_mode_snap_pending`.  After re-enable, every
`_enter_build` / `_enter_breakdown` / `_enter_climax` returns at 8337 for the
rest of the session.

Repro: `autovj/repro1.py` R2 (flag stays true, nothing queued, new attempts
queue nothing).  **Fix:** reset the flag next to `_drop_pending`.

### P1-3. An audio-clock jump stalls the render thread — VERIFIED

`beat_grid.py:2599-2606` / `5563-5570` (dense envelope) and `2541-2549` /
`5505-5513` (pulse envelope).  The envelope advance is a Python `while` loop
that writes one slot per 10 ms of clock delta, with no cap.

Measured for a single `update()`, v3 `dense_complex`: 84 ms for a 1-hour
jump, 1.74 s for one day, 5.3 s for three days (`beatgrid/t_clockjump.py`).

The clock is the audio manager's published time, which starts at `0.0`
(`unicornviz/audio/manager.py:153`) and becomes `time.monotonic()` (machine
uptime) at the first published block; `auto_vj.py:5413-5431` passes it
straight through.  A tracker that sees `0.0` once and then the real time
freezes the picture for uptime × 100 loop iterations.

**Not confirmed:** that a grid update really happens before the first
publish in a normal boot (the app was not launched).  The stall itself is
certain.  **Fix:** clamp the advance to the buffer length and re-base.

### P1-4. One non-finite audio value kills the drop signals — VERIFIED

`beat_grid.py:2060-2077` / `5024-5041` and four sibling sites.  Scalars are
read as `float(x or 0.0)`; NaN is truthy, so it passes, and the EMAs it
enters never recover.  After a single bad frame and 120 s of clean audio
(`beatgrid/t_nan.py`):

- NaN or Inf `bass_flux`: `drop_score` pinned at 0.000, `impact_novelty` NaN.
- NaN or Inf `bass_level_raw`: `drop_sustain`, the 90 s level ring and its
  low reference all NaN.
- Inf `spectral_flux`: `drop_score` 0.000, `drop_sustain` NaN.
- NaN or Inf `bass`: `drop_score` 0.000.

BPM and confidence survive.  The drop trigger, sustain and fizzle logic are
dead until restart.

The input is real: the core analyzer emits NaN `spectral_flux`, `bass_flux`
and `complex_onset_flux` after one NaN sample (core audit P2-4).  Either fix
alone closes the path; both should land.

**Fix:** sanitize each scalar once at the top of `update()`.

---

## P2

### P2-1. About 30 config keys cannot be set — VERIFIED

`auto_vj.py:3750-3756`, `2759-2762`.  The USER-mood override set only admits
keys that appear in some mood preset.  Keys read through `_profile_value()`
but present in no preset are therefore fixed at their code default:

- every `phrase_*` key: the eight phrase-bias multipliers, `phrase_bias_max`,
  the eight expected-bars values, and the proximity, tier, outro and
  neutral-bars settings;
- `drop_confirm_score`, `drop_fizzle_score`, `drop_fizzle_grace_s`,
  `drop_trigger_fastlane`, `hold_cool_slope`, `climax_extend_max_factor`.

With `profile = 'user'`, `phrase_bias_max = 0.5` and
`phrase_under_over_hold_mult = 0.1` in config, the controller still reads
0.15 and 0.8 (`autovj/repro1.py` R1).  The weights doc (lines 562-571) calls
these config keys.

**Consequence worth checking (SUSPECTED):** any replay experiment that
overrode one of these keys changed nothing, so a null result from it had no
leverage.  The replay harness was not read in this part of the audit.

**Fix:** for keys in no preset, let `_profile_value()` fall through to
`self._cfg`.

### P2-2. Track change ignores the hold states — VERIFIED

`auto_vj.py:5374-5379` → `5207-5249`; guards at `5540-5553`.  The
track-change effect swap runs before the USER-hold, PAUSE and startup-grace
early returns.  The first now-playing counter seen after start also counts as
a change, so a swap fires on the first enabled frame, inside the 90 s grace.

Repro: `autovj/repro2.py` (`goto_random_effect` in all three states, at
frame 1 and again 20 s later).  **Fix:** move the swap below the guards;
leave the phrase-clock reset where it is.

### P2-3. Deferred actions outrun their context (09-30 AV-3, wider)

`auto_vj.py:5431`, `8576-8674`, `8691-8717`.  Deferred drops and mode
entries fire inside `grid.update()`, ahead of every guard, and only
`toggle()` cancels them.  New this pass:

- The wait is unbounded.  Callbacks run only when downbeat confidence is at
  least 0.30 (`beat_grid.py:3649`), so a pending drop is held through any
  low-confidence stretch, then passes revalidation on scores captured when it
  was scheduled (8702-8706).
- The bass gate (8635-8674) adds up to four more bars.
- A deferred breakdown entry that fires during a USER hold calls
  `clear_postfx()` (8554).

**Fix:** one "cancel deferred actions" helper used on track change,
ping-pong entry and toggle; `_fire_drop` bails when the user is busy or the
source mode has changed.

### P2-4. Engagement counters count frames — VERIFIED

`auto_vj.py:6098-6107`.  While a BREAKDOWN landing is deferred to the
downbeat, `drop_fizzle_relative_count` / `_floor_count` increment every
frame (+7 in the 7 frames after one fizzle, `autovj/repro2.py` R7).  The same
pattern affects `_mode_blocked_by_source_count` (8453, 8530, 8890) and the
two `_drop_cruise_blocked_*` counters (5908-5910).  These values go into
corpus rows, so any analysis of "how often did this gate engage" reads
frame-rate-scaled numbers.

**Fix:** count only when no mode snap is pending.

### P2-5. One bad flux value blinds the dense envelope — VERIFIED

`beat_grid.py:2572-2597` / `5536-5597`.  A NaN or Inf `complex_onset_flux`
sits in the 800-entry normalizer history; the median goes NaN and every
z-score becomes 0 until it ages out.  At 60 fps the envelope is entirely
zero for 5.4 s and the first non-zero write comes 13.3 s later; at 30 fps,
18.8 s and 26.7 s (`beatgrid/t_misc3.py`).  The BPM holds meanwhile.

**Fix:** skip non-finite values before scoring and appending.

### P2-6. A stall leaves the grid offset — VERIFIED

`beat_grid.py:2340-2344` / `5304-5308` vs `3623-3660` / `6630-6667`.
`_advance_phase` is written to process every beat crossing after a hitched
frame, but its caller clamps the time step to 0.25 s, so the rest of the
stall is dropped.  After a 1.0 s stall at 120 BPM the grid advanced 0.5
beats instead of 2.0; after a 0.5 s stall it took 64 s to come back inside
tolerance (`beatgrid/t_misc.py` D, `t_misc3.py` D2).

**Fix:** when the delta is trusted audio time, advance by all of it.

### P2-7. Silence does not reset v3 — VERIFIED

`beat_grid.py:2349-2398` / `5313-5362`; `BeatTrackerV3` (7541-8073) has no
override.  `_reset_tempo_lock()` says the next song starts clean, but it
leaves `_v3_posterior`, the prime, the observation memo and the envelope
buffers.  After 60 s at 124 BPM and 30 s of silence, a 150 BPM track is
first published as 122.9 BPM at confidence 0.96 (`beatgrid/t_reset_v3.py`).
With the posterior reset patched into the test, the first reading is 132.6
at 0.08.  Time to the correct tempo was about 14 s either way; the defect is
the false confidence at the top of the track.

**Fix:** override the reset in `BeatTrackerV3`.

### P2-8. `top_candidates` goes stale under v3 — VERIFIED

`beat_grid.py:3206-3208` / `6213-6215`.  `_top_candidates` is written only at
the end of `_estimate_tempo_acf`, after the v2 gate stack accepts an update.
v3 sets the BPM from its posterior regardless.  Sixty seconds into a 150 BPM
track with low kick regularity: BPM 150.3 at confidence 1.00, but
`top_candidates` still reads 125.0 / 61.9 / 122.4 (`beatgrid/t_misc2.py` F).
The recommender (`auto_vj.py:7111`), the mood prime (7793) and the corpus
(6542) all read it.

**Fix:** compute it right after the raw observation, before any gate.

### P2-9. Frame counts stand in for seconds — VERIFIED

`beat_grid.py:2328-2331` / `5292-5295`, `1655` / `4619`, `7864`.  The ACF
runs every 8 calls, the flux normalizer window is 800 calls, and v3 applies
its observation once per call.

| fps | ACF cycles/s | normalizer window | 124→150 switch |
|---|---|---|---|
| 24 | 3.0 | 33.3 s | 8.5 s |
| 30 | 3.75 | 26.7 s | 8.2 s |
| 60 | 7.5 | 13.3 s | 5.6 s |
| 144 | 18.0 | 5.6 s | 4.3 s |

Every constant expressed in "cycles" therefore changes its real-time
meaning with the display rate, and with `[render] fps_limit`.  A detector
tuned at 60 fps is a different detector at 30.  (`beatgrid/t_fps.py`.)

**Fix:** drive these from audio time, for example per new 10 ms envelope
sample.

### P2-10. The Fire DJ overlay is never drawn — VERIFIED

`fire_dj_celebration.py:774-805`, `903-930`.  `render()` first runs the fire
simulation, which binds the simulation's own framebuffer, and then draws the
title and flames without re-binding the caller's target.  The composite
lands in the quarter-resolution fire texture (the one it is sampling), and
that framebuffer is still bound when the method returns.

Offscreen run, 4 s of the sequence: 0 of 230,400 target pixels touched; the
bound framebuffer after `render()` is the fire FBO (`mine/firedj_fbo.py`).
The app re-binds the screen for its own overlay pass in the normal case
(`app.py:7771`), so the HUD survives.

The timed side effects still fire (two bursts, three novas, five post-FX
slots, a scene swap, ping-pong), so the easter egg plays with no title, no
fire and no "Unlocked" text.

**Fix:** remember the bound framebuffer on entry (`ctx.fbo`) and `use()` it
again after the simulation step.

### P2-11 to P2-16

09-30 AV-4 through AV-9, unchanged.  See that audit and the table below.

---

## P3

| ID | Where | What | Fix |
|---|---|---|---|
| P3-1 | `auto_vj.py:7685-7691` | Switch-backoff decay subtracts the whole quiet time on each eval: a level-4 backoff clears at 120 s where one level per 90 s would hold it to 160 s — **VERIFIED** | Derive the level from the last apply |
| P3-2 | `auto_vj.py:9909-9913` | `trigger_grand_finale()` returns a message string, so the check is always true; with the finale drop-in absent Auto VJ logs "timed finale trigger fired" | Test a real boolean |
| P3-3 | `auto_vj.py:3994-4002` | `mode`, `manual_grace_s` and `_next_swap_min` are parsed and read by nothing; the grace actually used is core's 8.0 while all four presets say 3.0 | Wire or delete |
| P3-4 | `auto_vj.py:7225-7281`, `7436-7445` | The fold-admission block is unreachable: its four switches are only read through `getattr` defaults and nothing sets them | Parse the keys or remove |
| P3-5 | `auto_vj.py:5629-5635`, `8980-9031` | Ping-pong can be turned on with empty slots; mode sits in PINGPONG with the director skipped — **VERIFIED** | Refuse without both slots |
| P3-6 | `auto_vj.py:8700-8717` | With the split engine, a drop scheduled on impact or sustain is revalidated only against the legacy drop score (SUSPECTED whether intended) | Revalidate on the triggering signal |
| P3-7 | `auto_vj.py:5431`, `8228`, `8254`, `9894`, `4924` | `grid.update()` is unguarded inside `update()`; a raise skips the recommender, both corpus writers and the whole director that frame.  Four hint-bus conversions can do the same on a malformed value | Guard and sanitize |
| P3-8 | `auto_vj.py:6889-6993`, `5734-5737`, `7845-7875` | Per-frame waste: ZCR recomputed though `audio.zcr` exists, a centroid for a retired term, a 3,600-entry list with `pop(0)` for a feature that is off, a 130-key payload built each second and discarded when the log is closed | Remove |
| P3-9 | `auto_vj.py:3032-3069`, `1996-2006` | Log and corpus paths accept any location (`expanduser`, `mkdir(parents=True)`); the decision-log writer encodes and flushes per row on the render thread and swallows write errors | Confine; buffer; log failures |
| P3-10 | `auto_vj.py:3011`, `2278`, `7656` | Replay determinism: the controller RNG has no seed parameter; one row field and the user-busy gate read the wall clock instead of the injected one | Seed parameter; use the clock seam |
| P3-11 | `beat_grid.py:2313-2314` / `5277-5278`, `7854-7890` | With `env_source = pulses` (the code default, not the live setting) v3 never drops its lock in silence: BPM 124.6 at confidence 1.00 after 60 s — **VERIFIED** | Stamp and expire the held observation |
| P3-12 | `beat_grid.py:3482`, `3193`, `3620`, `3790` | v2 only: the primed-confidence floor never expires (0.900 ten minutes after one prime) — **VERIFIED** | Give the floor its own expiry |
| P3-13 | `beat_grid.py:2870` / `5834`, `3045` / `6052` | In dense envelope modes the density guard counts envelope samples, not onsets: 3,218 "per minute" for a 124 BPM click train — **VERIFIED** | Count rising edges |
| P3-14 | `beat_grid.py:8067-8069`, `7786`, `3426-3435` / `6433-6442` | v3 telemetry is wrong: fold-jump count misses decisive-rival snaps; `genre_evidence_applied_count` counts ticks, not cycles; `bars_since_lock`, the cold-start guard flag and the dwell-gated count never move — all **VERIFIED**, all logged to the corpus | Maintain them in v3 or stop logging them |
| P3-15 | `beat_grid.py:1485` / `4449`, `363`, `915` | `refractory_factor` is parsed and read by nothing; `_bar_phase_outliers` is written and never read; `_V2_ANALYSIS_REGION_CONFIDENCE_MIN` is defined twice (0.58, then 0.40) | Remove |
| P3-16 | `beat_grid.py:2286-2298` / `5250-5262` | Onset phase error is measured against the previous frame's phase, adding up to one frame of offset to the "mechanical offset" telemetry (not measured) | Extrapolate to the onset time |
| P3-17 | `beat_grid.py:7794-7799`, `7657-7679` | The v3 template matrix is built inside the first observation: one 14 ms frame about a second after start — **VERIFIED** | Build in `__init__` |
| P3-18 | `beat_grid.py:3759-3867` | Priming before the first `update()` is silently lost (expiry lands 1e9 s in the past); v2 `prime_tempo(nan)` would stick.  Latent: the caller guards | Initialize the clock; reject non-finite |
| P3-19 | `beat_grid.py:2913-2950` | The only unexplained v2 / v3-base difference: a cost-only memo was added to the "byte-identical protected" v2 copy and not to the live copy | Mirror it or revert it |
| P3-20 | `fire_dj_celebration.py:819-849`, `1009-1021`, `592-607`, `714-756`, `961-992` | Reaches `vj_api._app._auto_vj._pp_slot_a` to find its own controller; hard-coded Linux font paths instead of `unicornviz.fonts`; three shader compiles and two PIL rasters on the render thread at first trigger; sets post-FX slot 10 at the end and never restores the operator's slot; two dead helpers; fragment shader is 190 lines | Pass the controller in; use the shared loader; build at startup; restore the slot |
| P3-21 | `auto_vj.py` `HELP_ENTRIES`, `2387`, `3577`, `3743` | Ctrl+Alt+Shift+D (Fire DJ test) is handled but not listed; the Ctrl+J M text names three moods where the cycle has six; the MIDI action labelled "Cycle audio profile" cycles the mood | Correct the entries |

---

## Sync obligations (CLAUDE.md)

| Check | Result |
|---|---|
| `_VJ_WEIGHTS_DOC_VERSION` (114) = doc header | OK |
| Detector / Director / Recommender header lines | OK (rc.46 / rc.22 / rc.50) |
| ADR names the detector version | OK (`docs/adr/vj-system.md:10259`) |
| Doc table values vs code | OK for every listed constant |
| `__version__` vs README | **Mismatch**: code `1.0.0-rc.155`; README header (line 18) and newest changelog line (457) say rc.154 |
| Recommender weights in the doc | 12 of 13.  **Missing row:** `spectral_contrast_fit` |
| Director constants in the doc | **Missing:** the eight `_PHRASE_ROLE_BARS` values, `phrase_boundary_bar_unit`, `phrase_peak_flourish_min_cycle`, `phrase_outro_song_progress`, `phrase_track_change_neutral_bars` |
| Decider defaults in the doc | Ten match.  **Missing from the table:** the three `profile_auto_reco_switch_backoff_*` keys |
| Detector constants in the doc | **Missing:** 13 named constants (four `_V3_DENSITY_*` / `_V3_FOLD_SUSPECT_RATIOS`, nine `_V2_*`) and four inline defaults (`silence_reset_s`, `audio_dt_max_s`, `acf_score_floor`, `refractory_factor`) |
| Stale doc prose | Prefilter described as ±10 % (code: flat 4 BPM); no `tweaker` row in the ladder table; `drop_confirm_score` described as the fizzle check (code: fire-time revalidation) |
| `_RECO_WEIGHT_DEFAULTS` (training kit) | OK, all 13 present |
| `_DIRECTOR_CONSTANT_DEFAULTS` | **Missing names:** 13 `phrase_*` keys and all ten `profile_auto_reco_*` thresholds (in none of the three dicts) |
| `_DETECTOR_CONSTANT_DEFAULTS` | **Missing names:** 30, including eight `_V3_*`, both `_TACTUS_*`, twelve `_V2_*` gates.  These are invisible to the LLM scoring payload |
| Engagement counters for `_V3_*` gates | None for `_V3_ALT_SWITCH_COOLDOWN_BARS`, `_V3_FOLD_SUSPECT_GATE_MASS` (shares a counter), `_V3_OBS_FLOOR`, `_V3_FOLD_OBS_WEIGHT`, `_V3_PRIOR_GAIN`, `_V3_CONF_BAND` |

The full lists of missing names are in the readers' notes; ask and they go
into the doc-sync change.

---

## Status of the 09-30 findings

| ID | Status | Current lines (`auto_vj.py` unless noted) |
|---|---|---|
| AV-1 | Open (P1-1) | 7447-7484, 7667-7680 |
| AV-2 | Open, re-verified (P1-2) | 5286-5326, 8337-8374 |
| AV-3 | Open, wider (P2-3) | 5431, 8576-8717, 8131-8144 |
| AV-4 | Open | 8346, 8588, 8141, 8102-8125 |
| AV-5 | Open | 9568-9569 |
| AV-6 | Open | 9117, 9141, 9296; core `app.py:7390-7392` |
| AV-7 | Open | 9906-9918 |
| AV-8 | Open | 6742-6745 |
| AV-9 | Open | 4606-4680; MIDI label 3577 |
| AV-10 | Open | 5972-6034 |
| AV-11 | Open | 9167, overwritten at 8987 |
| AV-12 | Open | 9395-9400, 9440-9445, 9379, 9465, 9420 |
| AV-13 | Open | 3892-3920, 3143 |
| AV-14 | Open | 5286-5326 |
| AV-15 | Open | 6128 vs 5273; `_exit_drop` 8971-8973 |
| AV-16 | Open | `beat_grid.py` 579-583, 3653-3657, 6660-6664 |
| AV-17 | Open | 5375, 8710, 5945/5996, 1708-1712, 8131-8144 |
| AV-18 | Open | 1747-1801, 3038 |

Earlier detector audits (08-04, 08-13): the search-range clamp, lag
quantization, ACF overlap bias and unbounded pulse strength are fixed.  Still
present: the BPM-EMA alpha near its floor (`beat_grid.py:3467` / `6474`), the
clock-epoch mix (1983 / 4947; P1-3 is on the same path), and the pulse
written at the write head in the `pulses` path (2621-2660, deliberately
kept).

---

## Per-frame cost (detector)

6,000 synthetic frames per run:

| Engine / envelope | mean | p99 | max |
|---|---|---|---|
| v3 `dense_complex` (live) | 0.27 ms | 1.11 ms | 1.52 ms |
| v3 `pulses` | 0.18 ms | 1.00 ms | 2.02 ms |
| v2 `pulses` | 0.10 ms | 0.71 ms | 1.93 ms |

The detector is not a frame-budget problem.  One-time worst frame is P3-17
(14 ms).  The v2 decision layer that v3 computes and discards costs about
0.011 ms a frame.  The one avoidable per-frame allocation is
`_normalize_flux` (0.11 ms: `np.fromiter` over an 800-entry deque plus two
medians).

---

## Rule violations and design debt

- **Public Runtime Surface:** `auto_vj.py` reaches `app._audio_manager` at
  two sites (6707, 7024) though it already holds the manager, and reads
  seven `_v3_*` privates of its own detector.  `fire_dj_celebration.py`
  reaches through `vj_api._app` (P3-20).
- **Help entries:** P3-21.
- **Config paths within the project root:** P3-9.
- No `print()`, `global`, or module-level `random` / `np.random` in the
  drop-in.
- `beat_grid.py:311` is a 6,759-character single-line changelog comment whose
  rc.43-rc.37 tail is duplicated; about 30 % of `auto_vj.py` (2,968 lines) is
  dated changelog comments.  The ADR already holds this history.
- About 2,900 lines are duplicated between v2 and the v3 base by owner
  decision; P3-19 is the first drift between them.  The base's docstring
  still says "(v2)", and a "BeatTrackerV3 retired" comment sits directly above
  the live class.
- `toggle()` resets three things; about ten deferred or latched states
  survive it (the root of P1-2 and P2-3).
- There is no public reset or track-change entry point on the tracker;
  silence is the only reset path.
- `shutdown()` does not release the celebration's GL objects, the pinned
  pair or the auto-advance override.
- All five tuning harnesses in `tools/` construct the v2 `BeatTracker`; none
  exercises the live v3 engine.  They all still run.
- `install.sh` is clean: quoted paths, `set -euo pipefail`, list-form
  commands.  `--uninstall --root X` removes `X/drop-ins/auto-vj-01` without
  confirmation, as documented.
- The README links to a GitHub organization other than the one the
  repository is pushed to.

---

## What this audit did not do

- No real music, no live session, no judgment of detection accuracy or of
  any tuning value.
- `dense_flux`, v3 observation sources other than `template`, and
  `v3_obs_apply = 'cycle'` were not exercised; v1 was read only.
- 1,141 comment lines in `auto_vj.py` were not read.  `corpus_writers.py`
  belongs to training-kit-01 and is covered there.
- Did not fix anything.  P1-2, P1-4, P2-7 and P2-10 are each a few lines.
