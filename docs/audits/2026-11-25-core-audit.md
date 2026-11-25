# Core audit — `unicornviz/` (2026-11-25)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-25

Scope: the core package only (`unicornviz/`, 60 files, 46,267 lines, core
`1.0.0-beta.199`, master `c34ccdc`).  `drop-ins/`, `tools/`, `docs/` and
`tests/` were out of scope and were read only where a core finding had to be
confirmed against its caller.

Method:

- Every line of every core module was read, split four ways: `app.py`; the UI
  files (`overlays.py`, `vj_api.py`, `hotkeys.py` and eight small UI modules);
  the audio package, `remote_objects.py`, `midi.py`, `recording.py` and the
  thread helpers; and everything else (config, drop-in loader, stores, entry
  point, playlist, ANSI, GL helpers, core effects).  The 27 profile tables in
  `audio/profiles.py` (lines 470-3215) were checked by script for internal
  consistency instead of line by line; none was found inconsistent.
- Core test suite, ruff (bugbear, pyflakes, pylint, bandit and perf rule
  sets), bandit and pip-audit were run from a clean seat worktree.
- Findings marked **VERIFIED** were reproduced with a script.  The rest are
  from reading the code and name the exact lines.  **SUSPECTED** marks a part
  that could not be confirmed.  The three most severe `app.py` findings and
  five of the audio/UI findings were re-checked against the source or re-run
  by a second reader.
- Nothing was run against a real GL context, audio device or MIDI port, and
  the app itself was not launched.  Capture and MIDI logic was exercised
  against fake `sounddevice` / `rtmidi` modules.

Repro scripts: `/var/tmp/uv-core-audit/{app,ui,audio,mine}/`.

Severity:

- **P1**: can freeze or end a live show, or lose the owner's data.
- **P2**: wrong behavior an audience or the operator will notice, or a rare
  crash.
- **P3**: latent, cosmetic, or only reachable off the main path.

---

## Baseline

| Check | Result |
|---|---|
| Core tests | 3,240 passed, 4 skipped (73 s).  Green, and they miss every finding below. |
| bandit | 0 high, 0 medium, 89 low (try/except/pass, subprocess with list argv, `random` for visuals, one approved pickle).  None is a defect. |
| ruff bug rules | No undefined names or unused-variable defects.  378 broad `except Exception`, 11 `global` statements (see Rule violations). |
| pip-audit | **42 advisories, all against `Pillow 12.3.0`, all fixed in `13.0.0`.**  See Security findings.  Not remediated. |

---

## Summary

| ID | Sev | Area | One line |
|---|---|---|---|
| P1-1 | P1 | main loop | Turning on "Per-frame perf logging" in the config menu mid-session exits the app |
| P1-2 | P1 | main loop | Crash isolation is still partial: the post-FX chain, screen burst, controller-help renderer and eight other drop-in calls on the frame path are unguarded |
| P1-3 | P1 | effects | Ping-pong pinning still leaks effect instances; a crashed pinned effect is shown again and destroyed twice — **VERIFIED** |
| P1-4 | P1 | audio | One failed `stream.read()` leaves capture dead with no way back short of a restart — **VERIFIED** (trigger unconfirmed) |
| P2-1 | P2 | shutdown | Any exit before the keystroke logger is set up aborts teardown — **VERIFIED** |
| P2-2 | P2 | config | `transition_duration = 0` passes validation and divides by zero at the first transition — **VERIFIED** |
| P2-3 | P2 | config | A malformed `config.toml` exits with a raw traceback before logging exists — **VERIFIED** |
| P2-4 | P2 | audio | One NaN or Inf sample kills band levels and fills the FFT with NaN for the rest of the session — **VERIFIED** |
| P2-5 | P2 | recording | A long faststart rewrite is mistaken for a stall and ffmpeg is killed mid-rewrite — **VERIFIED** (size stays constant) |
| P2-6 | P2 | recording | The frame-rate cap aliases against the display: a 60 fps recording on a 60 Hz display gets about 40 unique frames/s — **VERIFIED** by simulation |
| P2-7 | P2 | recording | Odd window width or height makes libx264 exit at once — **VERIFIED** |
| P2-8 | P2 | audio | Source switches and helper recovery block the render thread (seconds; up to 300 s if PortAudio hangs) |
| P2-9 | P2 | audio | Auto-fallback only ever probes one candidate, and a stream that delivers nothing never counts as silent — **VERIFIED** |
| P2-10 | P2 | MIDI | An aux controller that is unplugged, or absent at open, is never retried — **VERIFIED** |
| P2-11 | P2 | IPC | Shared-memory segments are left in `/dev/shm` whenever the helper does not exit cleanly — **VERIFIED** |
| P2-12 | P2 | IPC | `SharedMemory(track=False)` still breaks on the Python 3.11 install paths |
| P2-13 | P2 | hotkeys | Rebinding a key kills or redirects the same action on MIDI pads and context slots — **VERIFIED** |
| P2-14 | P2 | hotkeys | Rebinds compare the raw modifier word: NumLock/CapsLock or the other Ctrl key defeats them — **VERIFIED** |
| P2-15 | P2 | hotkeys | One exception in a drop-in key handler unregisters it for the session |
| P2-16 | P2 | hotkeys | Ctrl+Shift+P changes the on-air preset instead of opening the presets modal — **VERIFIED** |
| P2-17 | P2 | overlays | Each custom CTA leaks a 2.2 MB texture — **VERIFIED** |
| P2-18 | P2 | overlays | Splash replay stops the main loop for 3 s and discards every input event |
| P2-19 | P2 | overlays | The ProjectM manager rescans the whole catalog about 30 times a frame — **VERIFIED** scaling |
| P2-20 | P2 | startup | Five drop-in calls at boot sit outside try/except; a raise aborts startup |
| P2-21 | P2 | Windows | Help links run `xdg-open` only |
| P2-22 | P2 | effects | Four core shaders still multiply `iTime` by a live audio value (Class A, carried over) |
| P3-1…P3-48 | P3 | various | See the P3 tables |

---

## P1

### P1-1. Perf-logging toggle exits the app

**Where:** `app.py:6809-6810`, `7952-7955` (setter `4624-4627`, menu row
`4366-4369`).

`perf_frame_start` is assigned at the top of the frame only when
`_perf_frames_enabled` is already true.  The default is false, so the local is
never bound.  When the operator turns the row on (keyboard or pointer), the
flag flips during event handling and the same frame then evaluates
`perf_after_subsystem_present - perf_frame_start` at 7955.  That raises
`UnboundLocalError` inside `run()`, which nothing catches.

The flag is saved, so the next launch starts with it on and works.  The crash
happens once, live, at the moment the operator asks for diagnostics.

**Fix:** read the flag into a local once at the top of each frame and use that
local for every perf block.

### P1-2. Crash isolation is still partial (carried over from 09-30 P1-4)

Fixed since 09-30: config-editor keys, MIDI dispatch, the late overlays, video
deck update/draw, the streamer write, transition-end destroy and first-effect
construction.

Still unguarded, each on the frame path, each ending the show on a raise:

- **Post-FX chain and screen burst** — `app.py:2885-2906` (`_apply_post_chain`,
  called at 8427/8476/8516/8571/8654) and `2756`, `8420`, `8436`.  `_render()`
  is called bare at 7754.
- **Controller-help renderer** — `overlays.py:1390-1392` (called from
  3976-3977); `overlays.render()` is called bare at `app.py:7775`/`7791`.
- `app.py:7728` — `self._auto_vj.training_badge()` every frame while the HUD
  is hidden (the default), with no `callable` check.
- `app.py:6982-6986`, `7189-7200` — post-FX scroll handlers and
  `toggle_auto_vj()` on wheel and middle click.
- `app.py:7074` → `3449-3458` — a context-menu click calls `hotkeys.handle()`
  unguarded; the keyboard path at 6928 is guarded.
- `app.py:7853` — `frame_fps_cap()`; `9805-9808` — `hosted_frame()` (mixer
  profile).
- `app.py:7211` → `2121-2193` and `6951` → `8771-8841` — display hot-plug and
  resize fan out to six drop-ins unguarded; a raise midway also leaves the
  FBOs at the old size.
- `app.py:6858` — a disk error while saving a profile on Enter.
- `app.py:7483` — `float()` on drop-in-supplied snapshot data.

**Fix:** route each through the existing `_call_guard` / `_event_guard`, and
wrap the overlay pass at its call site.

### P1-3. Ping-pong pinning leaks and double-destroys effects — VERIFIED (carried over from 09-30 P1-1)

**Where:** `app.py:10091-10105` (`cut_to_pinned`), `5870-5894`
(`_handle_effect_crash`), `10107-10123`.

Unchanged since 09-30.  `cut_to_pinned` overwrites `_current_effect` and
`_next_effect` without destroying them, so each ping-pong entry leaks the
previous effect and any in-flight transition target.  No `ctx.gc_mode` is set,
so their GL objects are never freed.

A second half was found this pass: a pinned effect that crashes stays in
`_pinned_pair`.  The next `cut_to_pinned` puts the destroyed instance on
screen, and `unpin_effect_pair` destroys it again.

Repro: `/var/tmp/uv-core-audit/app/repro.py` cases B and C (destroy counts 0
and 0 for the outgoing effects; destroy count 2 for the crashed one).  Whether
a second `destroy()` raises depends on the effect.

**Fix:** destroy outgoing current/next in `cut_to_pinned` when they are not
pair members; remove a crashed instance from the pair.

### P1-4. A failed audio read leaves capture dead for the session — VERIFIED

**Where:** `audio/capture.py:902-906`, `1277-1278`.

When `stream.read()` raises, the reader thread sets `_active = False` and
exits.  Every recovery path then refuses to act: `select_source`,
`cycle_source` and the auto-fallback all return `'inactive'` at line 1277, and
`AudioManager.start()` is only called at boot and on helper death.  Audio
reactivity stays off until the app is restarted.

Repro: `/var/tmp/uv-core-audit/audio/capture_fake.py` case b (fake
`sounddevice`).  **Not confirmed:** that PortAudio raises on a device replug
or PipeWire restart on this stack; the dead-end state itself is certain.

**Fix:** let `_switch_to_candidate_index` reopen from the inactive state, and
have the analysis loop retry when `active` goes false.

---

## P2

### P2-1. Early exits abort teardown — VERIFIED

`app.py:8088` reads `self._keystroke_logger` outside a guarded shutdown step,
and the attribute is first set at 6437.  Any shutdown before that point raises
`AttributeError` with `_shutdown_complete` already true, so the rest of
teardown never runs: audio manager stop, audio helper close, MIDI stop,
runtime-state save, GL/SDL teardown.

Reached by: Esc on the splash, a failed required audio start (6183), a bad
`[midi] cc_map` key (6304), "No effects found" (6318).  In `__main__.py` the
`AttributeError` from the `finally` at 677-680 also replaces the original
exception, so the "audio capture failed" help text never prints.

This is what remains of 09-30 P2-5 (the double teardown itself is fixed).
Repro: `app/repro.py` case A.  **Fix:** set the attribute to `None` in
`__init__` and keep every statement of `_shutdown_runtime` inside `step()`.

### P2-2. `transition_duration = 0` crashes the first transition — VERIFIED

`app.py:738-740` stores the value uncast and unclamped; `8542-8546` divides by
it.  `Config.validate()` accepts 0.  On a first run the first transition
raises `ZeroDivisionError` in the unguarded `_render`.  Later boots are masked
because the saved profile re-applies the value through a clamping setter.

Related, all accepted by validation (VERIFIED, `mine/repro.py`):
`[audio] fft_bands = 0`, `[demo] effect_duration = -5`, `[window] width = 0`.

**Fix:** clamp in `__init__`; add range checks to `_extra_constraints`.

### P2-3. A malformed `config.toml` exits with a raw traceback — VERIFIED

`__main__.py:657` constructs `Config` outside any `try`, before logging is set
up.  `tomllib.TOMLDecodeError` reaches the terminal as a traceback; under
`pythonw` on Windows there is no terminal and the app simply does not start.
The classic trigger is a Windows path in double quotes
(`directory = "C:\Users\me"` → "Invalid hex value").

Also VERIFIED: `window = 5` (a scalar where a table belongs) raises
`AttributeError` in `_extra_constraints` (`config.py:254`) instead of a
validation error.

**Fix:** catch `TOMLDecodeError` in `main()`, print the line and column, exit
2; return early from `_extra_constraints` when a section is not a table.

### P2-4. One NaN sample poisons the analyzer — VERIFIED

`audio/analyzer.py:859`, `875`, `996-998`.  A NaN or Inf sample makes `energy`
NaN, the `energy <= 1e-5` gates are false, and `_smoothed *= 0.75; += spectrum`
keeps NaN from then on.  `data.fft` is all NaN (uploaded to shaders) and
bass/mid/treble read 0.0 for the rest of the session.  No exception is raised,
so the analysis-thread guard added for 09-30 P1-5 never fires, and its
`set_profile` reset does not clear `_smoothed` anyway.

Repro: `audio/analyzer_checks.py` (42 s of clean audio after one NaN sample:
512 of 512 bins NaN).  **Fix:** `np.nan_to_num` the block at the top of
`process()`.

### P2-5. A long faststart rewrite gets ffmpeg killed — VERIFIED (constant size)

`recording.py:1048-1083`.  `_await_finalize` treats an unchanged `st_size` as
no progress.  During ffmpeg's `+faststart` second pass the file size does not
change (measured on a 1.79 GB file).  After 20 s the code sends SIGINT, and
after 20 s more it calls `process.kill()` mid-rewrite, which leaves no moov
atom and an unplayable file.

Needs a rewrite longer than 40 s: roughly 10 GB on a SATA disk, much more on
NVMe.  For a recording on a slow or network disk this is the whole set lost,
which would be P1.  **Fix:** count `st_mtime_ns` changes as progress, or drop
faststart for long recordings.

### P2-6. Recording frame-rate cap aliases against the display — VERIFIED by simulation

`frame_tap.py:51-55`, `128`.  The cap tests `now - last >= interval` and then
stamps `last = now`.  With the cap equal to the refresh rate, frame-time
jitter makes about every third frame miss the test.  Simulated delivery at a
60 fps cap: 40 fps on 60 Hz, 37.5 on 75 Hz, 48 on 144 Hz.  The writer
duplicates frames to fill, so recordings judder.

**Fix:** advance `last_delivery_t` by `interval` (accumulator with a clamp).

### P2-7. Odd window sizes fail under libx264 — VERIFIED

`recording.py:776-853`.  The window size goes to ffmpeg unchanged with
`-pix_fmt yuv420p`.  libx264 at 1281×721 exits with rc 187 and an empty file;
the operator sees "RECORDING FAILED".  Hardware encoders and libopenh264 on
this box accepted it.  **Fix:** add an even-size crop or pad filter.

### P2-8. Audio calls block the render thread

- `audio/process.py:94-100`, `remote_objects.py:1132-1145`: `select_source`,
  `cycle_source`, `start`, `stop` and `get_recent_pcm_window` are waited calls
  with a 300 s ceiling, issued from the render thread (`app.py:9450`/`9455`).
  A source switch costs up to 3 s for the reader join plus 3 s for the close
  plus the PortAudio open.  If the open hangs (the documented 2026-09-14
  OBS/JACK case) the picture freezes for up to five minutes.
- `app.py:9307-9322`: helper-death recovery calls `audio_manager.start()` with
  a 4 s timeout on the render thread.  It also restarts with boot-time state:
  the analyzer keeps the boot profile while the HUD names the session's
  (09-30 P3-3, still open), and the capture reopens the boot-time source and
  overwrites the operator's saved source choices (`audio/capture.py:448`,
  `753`).

**Fix:** issue switches without waiting and report through derived state;
recover on a worker thread and re-apply profile and source state first.

### P2-9. Auto-fallback cannot reach a working source — VERIFIED

- `audio/capture.py:957-965`, `1043`: the fallback probes only the first
  viable candidate after the current one, because the index moves only on a
  successful switch.  With candidates [silent (current), silent, playing] the
  third is never tried (142 probes, all of the same device).
- `audio/capture.py:898-900`, `943-947`: silence is counted in blocks
  received, so a stream that delivers no data at all is never silent and
  fallback never starts.  Whether Pulse/PipeWire sources stall this way in
  practice is not confirmed.

Repro: `audio/capture_fake.py` cases a and c.  **Fix:** a rotating probe
cursor; silence measured in wall time.

### P2-10. Aux MIDI devices are never retried — VERIFIED

`midi.py:458-465`, `627-637`.  An aux device that disappears is removed and
retried exactly once, at the moment it is known to be gone.  The hint is
stored nowhere, so `maintenance_update` returns early from then on.  A
controller unplugged mid-set, or absent when the mixer opens, stays dead until
restart.  This is the core half of the 09-30 hardware finding "no MIDI
reconnect after unplug".

Repro: `audio/midi_fake.py`.  **Fix:** keep a set of wanted aux hints and
retry them in `_maintain_aux`.

### P2-11. Shared memory leaks on any unclean helper exit — VERIFIED

`remote_objects.py:303`, `315`, `351-358`, `1098-1102`.  Segments are created
untracked and unlinked only in the exporter's `close()`, which runs only on a
clean exit.  A helper segfault, SIGKILL, SIGTERM/SIGHUP (only SIGINT is
ignored), or the 3 s kill in `RemoteHost.close()` leaves every segment in
`/dev/shm`, which is RAM-backed, until reboot.  For the mixer that is decoded
tracks and stems.

Repro: `audio/ro_test.py` step 3.  **Fix:** have the client record attached
segment names and unlink them when the helper dies, or sweep by prefix at
startup.

### P2-12. `SharedMemory(track=False)` on Python 3.11

`remote_objects.py:303`, `315`, `404`.  The argument is 3.13+.  Release
bundles now default to 3.14.7, but 3.11 is still reachable:
`fetch_runtime.sh` default flavor, aarch64 or no-wheelhouse installs
(`tools/install/lib.sh:196-205`), `tools/install_windows.ps1`,
`install_linux.sh --system-python`, and `pyproject.toml` (`>=3.11`).  There,
any published array of 1 MiB or more raises `TypeError`; core audio publishes
nothing that large, so the mixer is what breaks.  Carried over from the 09-30
Windows audit.

**Fix:** pass `track=False` only on 3.13+, and unregister from the resource
tracker otherwise; or raise `requires-python`.

### P2-13. A keyboard rebind breaks the same action elsewhere — VERIFIED

`hotkeys.py:564-567`, `669-675`, `1596-1598`, `239-268`.  MIDI pads,
performance context slots and the context menu trigger an action by replaying
its default key chord through `handle()`.  After a rebind the override logic
treats that default chord as freed.  With `next` moved to J, a pad mapped to
`next` does nothing; if `random` is then bound onto the freed N, the pad fires
`random`.

Repro: `ui/rebind.py` cases 7-9.  **Fix:** dispatch named actions by name
instead of by replayed chord.

### P2-14. Rebinds compare the raw modifier word — VERIFIED

`hotkeys.py:257-266`.  The SDL modifier word is compared exactly and is never
masked anywhere in core.  With NumLock or CapsLock on, a rebound key does not
fire and the old default is not freed.  An override captured with left Ctrl
does not match right Ctrl.  Defaults that carry Shift (0x3) never equal a real
press (0x1 or 0x2).  Conflict detection uses the same compare and finds
nothing with NumLock on.

Together with P2-13 this is what remains of 09-30 P2-7.  Repro: `ui/rebind.py`
cases 2-6.  **Fix:** normalize the modifier at capture, at load and in
`translate_override_chord`.

### P2-15. One exception unregisters a drop-in's keys

`hotkeys.py:990-996`.  A drop-in key handler that raises once is unregistered
for the rest of the session.  A transient error (a `None` during a track load)
silently removes every hotkey that drop-in owns; the only sign is a log line.
**Fix:** throttle the log and keep the handler, as the MIDI path does.

### P2-16. Ctrl+Shift+P is unreachable on preset effects — VERIFIED

`hotkeys.py:915-943` vs `1640-1641`.  The effect-local Ctrl+N/P/R block tests
only for Ctrl, so on ProjectM and the sim showcase Ctrl+Shift+P calls
`prev_preset()` and returns.  The on-air preset changes and the presets modal
never opens.  Repro: `ui/ctrl_shift_p.py`.  **Fix:** require Shift to be up.

### P2-17. Custom CTAs leak textures — VERIFIED

`cta_overlay.py:91-96`, `252-259`.  `trigger_custom()` sets
`self._textures = None` without releasing; the release in `_ensure_resources`
only runs when the list is not `None`.  Each custom CTA leaks one 1600×340
RGBA texture (2.2 MB), the first also leaks the three default slots.  Twenty
custom CTAs left 23 live textures, about 48 MB (`ui/cta_leak.py`).
**Fix:** release the old list in `trigger_custom`.

### P2-18. Splash replay freezes the loop for 3 s

`splash.py:230-277`, `vj_api.py:722-724`, `app.py:6043`.  `Splash.run()` is
its own loop.  During a replay nothing else runs: no MIDI, no subsystem
updates, and every SDL event except `SDL_QUIT` is polled and dropped,
including input to the mixer and Control Room windows.  The returned quit
flag is ignored.  Triggers: the `u` key (through unicorn-tears-01), the MIDI
`replay_splash` action, the context menu.  Whether recorder and stream frames
also stop was not traced; the frame loop is not running, so they likely do.

**Fix:** run the replay as an overlay state inside the normal frame loop.

### P2-19. ProjectM manager rescans the catalog every frame — VERIFIED scaling

`overlays.py:4416-4430`, `5076`, `5124`, `5135`, `5166`;
`catalog_browser.py:97-121`, `187-218`.  `filtered()` is uncached and called
about 30 times a frame while the manager is open.  Python-side cost against a
no-op GL mock: 2.6 ms at 500 presets, 4.7 ms at 2,000, 18.7 ms at 10,000.
`modal_snapshot()` adds 3.9-5 ms at 10,000 for the Control Room.  The real
library size was not available in the audit checkout.

**Fix:** cache `filtered()` and per-category stats, invalidated on change.

### P2-20. Startup drop-in calls outside try/except

Drop-In Independence rule 3.  A raise in any of these aborts boot and then
hits P2-1: the multihead constructor (`app.py:866-874`),
`_webcam_system.start()` and the `cycle_interval` cast (`2491-2492`),
`set_vj_api` (`2498`, `2526`), streamer `set_vj_api` / `start()`
(`6703-6712`).  **Fix:** extend the existing try blocks and fall back to the
null controllers.

### P2-21. Help links are Linux-only

`app.py:9019-9033`.  `_open_external_url` runs `['xdg-open', target]` with no
Windows or macOS branch, so on Windows every help-rail link reports "failed
to open link".  **Fix:** `webbrowser.open`.

### P2-22. Core shaders still scale `iTime` by audio (Class A, carried over)

`effects/audio_spectrum.py:139`, `audio_waveforms.py:102`,
`audio_tracks.py:105`, `audio_spectrogram.py:85`.  `self.time` starts anywhere
in 0-10,000 s, so `iTime * (k + iBass * 0.2)` moves the phase by up to 2,000
radians for a 1.0 change in bass.  The nebula, pulse and sweep patterns jump
on every level change instead of speeding up.  `audio_bass_machine.py:668-671`
already does it correctly (audio modulates the rate of an accumulated phase).

**Fix:** accumulate the phase in `update()` and pass it as a uniform.

---

## P3

### Main loop and app state (`app.py`)

| ID | Where | What | Fix |
|---|---|---|---|
| P3-1 | `5074-5086` vs `5066-5072` | "Capture latency" row writes runtime state only while the row reads `cfg`; the row never moves and HIGH cannot be reached from LOW — **VERIFIED** | `cfg.set_override` as the sibling setters do |
| P3-2 | `10759-10761`, `10941-10942` | Effects-browser pin and ProjectM-only set the effect lock even when the switch was refused; the show sits on the old effect while the flash says "pinned" | Lock only when the requested class is on screen |
| P3-3 | `7923-7926`, `8014-8021` | `_call_guard` disables a step for the whole session and is never reset; a stream can show LIVE while sending no frames (that `write_frame` raises is unconfirmed) | Reset on stream start and overlay toggle; flash a message |
| P3-4 | `7946-7951`, `9185-9191` | Mixer profile: a closed console re-opens the exit confirmation every frame after "No" (trigger unconfirmed) | `force=True` |
| P3-5 | `8790-8813`, `8965-8968`, `2175-2178` | Resize skips the off-screen pinned effect; Rainbow Nova and the dancing unicorn are never resized | Add to the fan-out |
| P3-6 | `8057-8147` | Shutdown never releases the FBO pair, blend program, four auxiliary FBOs, three overlays, the off-screen pinned effect or `_midi_out` | Add to the step list |
| P3-7 | `6304-6305`, `899-911`, `10508-10522` | `int(k)` on `[midi]` map keys and on hand-edited runtime-state keys is unguarded (09-30 P3-10, half open) | Skip bad keys with a warning |
| P3-8 | `9892-9928`, `9501-9513` | The first frame of a new recording or stream is the last frame read during the previous one | Clear `_stream_pbo_primed` when consumers go 0 → 1 |
| P3-9 | `6824-6828`, `6949-6964` | Closing the main window with a second window open is ignored (SUSPECTED; relies on SDL's last-window `SDL_QUIT`) | Treat CLOSE on the main window as exit |
| P3-10 | `4910-4918` | `pactl` runs on the render thread the first time the Recording tab opens, then is cached for the session | Enumerate off-thread; refresh on open |
| P3-11 | `1031-1202`, `1613-1620`, `1437`, `1449`, `3318-3321` | Hint buses and `get_frame_capture()` have no locks (SUSPECTED: which callers run off the main thread) | Copy before iterating; return one tuple |
| P3-12 | `1499-1507` | `TypeError` arity retry re-runs a renderer that raised `TypeError` itself (09-30 P3-7, open) | Decide arity once |
| P3-13 | `5702-5735` | `_switch_effect` unpins before the refusal returns (09-30 P3-8, open) | Move the unpin down |
| P3-14 | `1505`, `1581`, `7280-7363`, `7878-7926` | Twelve per-frame `log.warning` sites are still unthrottled (09-30 P3-6, partly fixed) | Use `CallGuard` |

### UI files

| ID | Where | What | Fix |
|---|---|---|---|
| P3-15 | `vj_api.py:1747`, `1771` | `operator_panels()` / `operator_pages()` iterate the live dict from the Control Room thread; 1,144 `RuntimeError`s in 3 s under a writer — **VERIFIED** (09-30 P3-13, open) | `list(...)` |
| P3-16 | `overlays.py:1090-1106` | Login/logout rail icon shows "logout" when not logged in, because the test is string-matching free text — **VERIFIED** | Publish a boolean |
| P3-17 | `overlays.py:5848-5949`, `6168-6195` | Help overlay re-wraps every section each frame: 3.4 ms at 1080p, 8.1 ms at 4K — **VERIFIED** | Cache the layout |
| P3-18 | `hotkeys.py:1104-1226` | With the audio selector, MIDI selector or webcam editor open, unhandled keys fall through to global dispatch (Delete disables the on-air effect) | Return unless ESC or the modal's toggle |
| P3-19 | `hotkeys.py:989-1000` before `1083-1101` | The tour does not get first refusal on keys | Move the tour branch up |
| P3-20 | `overlays.py:471-556`, `4602-4616` | Help, tour hints and tooltips show default chords after a rebind | Push effective chords to `Overlays` |
| P3-21 | `overlays.py:2846-2853`, `2416-2422` | A revealed secret row stays revealed across close/reopen on the audience window (whether it reaches a stream is unconfirmed) | Clear on close and tab change |
| P3-22 | `vj_api.py:932-936` | `sync_audio_selector()` drops divider rows; headings render as devices — **VERIFIED** | Pass the dividers |
| P3-23 | `overlays.py:501`, `hotkeys.py:189`, `199` | `u` "Replay splash" has no core handler (works only with unicorn-tears-01); the `ansi` action maps to `a`, which opens the audio selector | Handle `u` in core; retire `ansi` |
| P3-24 | `overlays.py:100-107` | Decomposed (NFD) text draws `?` after each accented letter — **VERIFIED** | NFC-normalize first |
| P3-25 | `hotkeys.py:863-881` | Preset names and search boxes accept only `a-z 0-9 space - . /` | Use the text-input path |
| P3-26 | `hotkeys.py:776-788` | ProjectM manager capability check omits six members it later uses | Complete the list |
| P3-27 | `now_playing.py:30-32`, `99-101` | Hub docstring says render-loop only; the Control Room thread calls it and can hit a dict-size change | Snapshot the items |
| P3-28 | `vj_api.py:1873-1877` | `fire_midi_action` logs handler exceptions at DEBUG and returns True | Throttled warning |
| P3-29 | `overlays.py:1525`, `3745`, `4080`; `cta_overlay.py:61`; `now_spinning.py:47` | HUD, tour, selectors, presets, browser, CTA and Now Spinning are fixed-pixel: quarter size at 4K, and the CTA overflows a 1280 window | Scale with resolution |
| P3-30 | `overlays.py:4362`, `5911-5917`, `5351`, `6482`, `1763…6152`, `7390-7516` | 09-30 UI-audit items still open: webcam editor capped at 12 rows, help cards dropped, FRAME MS shows `33 − ms`, salted `hash()` colors (**VERIFIED**), 8 px centering, dead CTA copy | As recorded on 09-30 |

### Audio, IPC, MIDI, recording

| ID | Where | What | Fix |
|---|---|---|---|
| P3-31 | `remote_objects.py:969-970`, `1003-1028` | `_apply_state` unguarded in `_receive`: an exception kills the receiver with `alive` still true, waits time out and recovery never fires — **VERIFIED** (09-30 P3-4, open) | try/except per message; cleanup in `finally` |
| P3-32 | `remote_objects.py:950-982` | A helper that dies at once stalls startup for the full 20 s spawn timeout — **VERIFIED** | Set `_ready` on receiver exit |
| P3-33 | `midi.py:533-554`, `638-648` | Partial open failure leaks the opened ports; the 2 s retry leaks about two ALSA clients per attempt — **VERIFIED** with a fake (09-30 P3-5, open) | Destroy `opened` in the except branch |
| P3-34 | `audio/capture.py:844-853`, `1303-1319` | A refused switch leaves capture dead with `active` true; the next switch closes the stream under a reader that may still be in `read()` — **VERIFIED** (09-30 P3-1, worse than recorded) | Keep the thread reference; refuse while alive |
| P3-35 | `audio/analyzer.py:411-428` | `set_sample_rate` does not rebuild band slices or flux weights; at 44.1 kHz and 96 kHz the bands are wrong for the session — **VERIFIED** | Call `_setup_frequency_bands()` |
| P3-36 | `recording.py:1119-1124`, `959` | Software retry still only runs when `Popen` raises; a hardware encoder that fails late is picked again every time (09-30 P3-9, open) | Invalidate the cache and restart once |
| P3-37 | `recording.py:695-701`, `754-757` | Windows: `ffmpeg -list_devices` (10 s timeout, uncached) on the render thread on every record start; first record press runs encoder probes synchronously | Cache; move to a worker |
| P3-38 | `audio/capture.py:972-976` | The fallback probe passes the parent's device index to a fresh process; after a hot-plug it measures a different device | Pass the name |
| P3-39 | `workers.py:106-108` | Quitting during library analysis hangs the exit until the running job ends — **VERIFIED** (6.12 s for a 6 s job) | Terminate workers on shutdown |
| P3-40 | `audio/manager.py:412`, `545`, `769`; `remote_objects.py:1019-1024`, `1085`; `963-967` | `process()` outside `_analysis_lock` (09-30 P3-2, open); `_shield` unsynchronized; one undecodable delta drops the whole state message | Assign tables once; lock; decode per path |
| P3-41 | `audio/manager.py:293-341`; `midi.py:745-748` | `stop()` can terminate PortAudio during an in-flight fallback open (SUSPECTED); `MidiOut.open` throttle also blocks a second port after a success — **VERIFIED** | Skip terminate when unclean; stamp on failure only |

### Config, loader, stores, entry point, ANSI, effects

| ID | Where | What | Fix |
|---|---|---|---|
| P3-42 | `config.py:226-234`, `274-285` | A numeric `audio.latency` is documented (`docs/configuration.md:118`, the error text itself) but always fails the type check — **VERIFIED** | Exempt the key from the default-type check |
| P3-43 | `dropins.py:121-123`, `172-207` | `[dropins] exclude` is read from `APP_ROOT/config.toml` only; `--config other.toml` is ignored for excludes — **VERIFIED** | Take the excludes from the loaded `Config` |
| P3-44 | `playlist.py:42-43` | `[playlist] sequence` matches class names only, so display names with spaces are silently ignored and the full playlist is used — **VERIFIED**.  `--sequence` help and the module docstring say display names work; `start_effect` accepts both | Match `NAME` or class name |
| P3-45 | `__main__.py:329-331`, `664-665`; `517`; `373` | The settings-quarantine ERROR is logged before file logging exists, so the log never records it; no `threading.excepthook`, so worker-thread crashes never reach the log; one log file per run with no retention (216 files, 834 MB in the live `logs/`) | Reorder; set the hook; prune |
| P3-46 | `ansi/loader.py:136-224`, `79-90`; `ansi/renderer.py:36-56` | Cursor-forward past the canvas width makes `canvas_to_texture` raise (the viewer catches it and skips the art) — **VERIFIED**.  SGR 5 (iCE bright background) is ignored although the docstring claims support — **VERIFIED**.  A SAUCE record with no preceding SUB eats one content byte — **VERIFIED**.  SAUCE width/height and cursor position are unbounded (2000×3000 → 6 M cells, 2.5 s).  Largest bundled art costs 390 ms on the render thread to load — **VERIFIED** | Clamp columns and sizes; handle SGR 5; strip 128 then an optional SUB; vectorize the blit |
| P3-47 | `effects/audio_chromogram.py:68`, `508-567`; `audio_spectrogram.py:113`; `audio_centroid.py:40` | Effects hard-code 48 kHz (centroid: 44.1 kHz) while capture uses the device rate; at 44.1 kHz the chromogram's note readout is 1.47 semitones off.  Chromogram prefill costs 23 ms on the render thread at each activation — **VERIFIED** | Read the rate from the audio manager; prefill off-thread or in slices |
| P3-48 | `effects/registry.py:56-59`, `110-128`, `131-133`; `effects/audio_sine.py:283-364`; `audio_waveforms.py:198-270`; `audio_spectrogram.py:194-208`; `audio_tracks.py:270-273`; `audio_centroid.py:226` | Pack detection uses `/` so every drop-in effect reports pack `core` on Windows — **VERIFIED**.  Abstract intermediate subclasses would be registered as effects.  Empty registry (mixer profile) re-runs discovery on every call.  Audio Sine and Audio Waveforms build vertex lists in Python inside `render()`.  Spectrogram and Tracks scroll once per frame, so the time span depends on FPS.  `glow_width` is declared and never used | Use `Path.parts`; skip abstract classes; cache an empty result; preallocate; fixed time base |

---

## Security findings (bandit / pip-audit) — reported, not remediated

**pip-audit: 42 advisories, one package.**

| Package | Pinned | Fixed in | Advisories |
|---|---|---|---|
| Pillow | 12.3.0 | 13.0.0 | 42 (19 PYSEC-2026-43xx, 22 CVE-2026-xxxxx, GHSA-78g4-xqwv-q3g8).  Full list: `/var/tmp/uv-core-audit/pip-audit.txt` |

- `requirements.txt` pins `Pillow==12.3.0` exactly; `pyproject.toml` asks for
  `Pillow>=10.0`.  Moving to 13.0.0 conflicts with no other pin in
  `requirements.txt`.
- It is a major-version bump.  Core uses `ImageFont.truetype`,
  `ImageFont.load_default(size=)`, `ImageDraw.textbbox` / `textlength` /
  `multiline_textbbox`, `Image.new`, `ImageColor.getrgb` and image `tobytes()`
  (`fonts.py`, `gpu2d.py`, `overlays.py`, `cta_overlay.py`,
  `now_spinning.py`, `splash.py`).  None of these was tested against 13.0.0
  in this audit.
- Exposure was not assessed per advisory.  The app decodes image files from
  the owner's own folders and, through drop-ins, cover art embedded in music
  files.

Per the project rule this is left for the owner: whether and when to upgrade.

**bandit: 89 low, no medium or high.**  All are false positives or approved:

- B603/B607/B404 (16): subprocess calls with list argv and no shell; device
  indices are passed through `int()`.
- B110/B112 (about 65): `try/except/pass` in cleanup and best-effort paths.
- B311 (2): `random` for visual variation.
- B403/B301: `pickle` in `remote_objects.py:829` between the app and its own
  helper process, already carrying `# nosec B301` with an owner approval note
  dated 2026-09-24.

**Manual review:** no `eval`/`exec`, no `shell=True`, no command built from a
string, no runtime network access anywhere in core.  MIDI data is logged at
DEBUG only.

---

## Rule violations (CLAUDE.md)

- **Config paths must stay within the project root.**  `paths.resolve_path`
  (`paths.py:44-53`) returns absolute paths unchanged and does not contain
  `..`.  Affects the splash image, ANSI directory, log directory and recording
  directory.  The config menu's own hints say "or absolute", so the rule text
  and the code disagree; one of them should change.
- **No `global` state.**  `__main__.py:436`, `482`; `dropins.py:179`;
  `hotkeys.py:320`, `331`; `recording.py:112`, `153`, `994`;
  `gc_tuning.py:137`; `x11_errors.py:93`.  Also the `VJApi._current`
  singleton (`vj_api.py:81-85`) and the module-level effect registry.
- **Public Runtime Surface.**  `hotkeys.py` reaches into private state at
  about 14 sites, for example `2088-2089` (`a._auto_advance`, where
  `vj_api.set_auto_advance` exists), `1825` (`# noqa: SLF001` outside an owner
  module), `764` and `525-727` (Auto VJ `_ctrlj_armed`, `_grid`, `_mode`).
  `app.py` reads private fields of Auto VJ, multihead, Control Room, the
  recorder and `Overlays` at about 35 sites (`1001-1004`, `1654`,
  `4925-5059`, `7324-7438`).  `vj_api.py` carries 144 `SLF001` suppressions
  (allowed as an owner module) and also writes drop-in privates
  (`2394-2409`).
- **Keys documented outside the help entries.**  Ten modal footers hardcode
  key hints (`overlays.py:3129`, `3412-3416`, `3834`, `4151`, `4264`,
  `4384-4386`, `4849-4851`, `5028-5033`, `5190-5211`, `5295`), which is also
  why they go stale after a rebind (P3-20).
- **Help-icon state rule.**  Violated by P3-16.  The size-bucket and
  no-resample rules are followed.
- **Core must not carry drop-in specifics.**  `overlays.py:6753-6754`
  (`'akai_apc_mini_mk2'`); `hotkeys.py:883-888`, `1126-1209` (webcam editor
  keys); `app.py:5650`, `6195-6222`, `6321-6343`, `7385`, `9089` (effect
  names `ProjectMEffect`, `Image Showcase`, `Sim Showcase`, `ANSIViewer`);
  `effects/audio_chromogram.py:64` (an effect importing `VJApi`).
- **GLSL conventions.**  Fragment shaders over 150 lines:
  `audio_bass_machine.py` (535) and `audio_chromogram.py` (215).  The
  stage/uniform comment block is missing from 24 of 26 shader strings in the
  core effects and from the shaders at `app.py:2631-2827`, `3067-3183` and
  `secondary_gl_window.py:102-123`.
- **`render()` must not allocate.**  See P3-48; also per-frame list builds for
  array uniforms at `audio_chromogram.py:636` and `audio_bass_machine.py:695`.
- **Cross-platform.**  `app.py:9026` (P2-21); `4578-4584` (`venv-ft/bin/python`
  and XDG paths, no Windows branch); `22-23` (Wayland forced on macOS);
  `2219-2221` (x11 fallback on every platform); no `CREATE_NO_WINDOW` on the
  four ffmpeg calls in `recording.py`, so `pythonw` flashes consoles.
- **Minor.**  `print()` in `__main__._self_test` (deliberate CLI output);
  `%` formatting at `audio/manager.py:694-698`; three one-line `__init__.py`
  files without the `__future__` import; double-quoted strings throughout
  `config.py` and `playlist.py`; `Config` and `Playlist` have no class
  docstring.

---

## Status of the 09-30 core findings

| 09-30 ID | Status now |
|---|---|
| P1-1 ping-pong leak | **Open** (P1-3) |
| P1-2 recording stop blocks render | Fixed (`recording.py:1224-1326`); new issue in the finalizer (P2-5) |
| P1-3 stalled encoder hangs `stop()` | Fixed (`recording.py:1261-1304`) |
| P1-4 crash isolation | Partly fixed (P1-2) |
| P1-5 analysis thread unguarded | Fixed (`audio/manager.py:365-387`); does not cover P2-4 |
| P1-6 unreadable stores overwritten | Fixed (`safe_store.py`, all three stores) |
| P2-1 `set_override` corrupts defaults | Fixed (`config.py:410-423`) |
| P2-3 large blocks truncate the FFT | Fixed (`audio/analyzer.py:845-852`, re-verified) |
| P2-4 overlays redraw outgoing effect | Fixed (`app.py:8607-8670`) |
| P2-5 Esc during splash | Double teardown fixed; teardown now aborts instead (P2-1) |
| P2-7 rebind frees old key / conflicts | Partly fixed (P2-13, P2-14, P3-20) |
| P2-8 Class A `iTime` | Open in four core shaders (P2-22) |
| P2-10 two threads switch streams | Fixed (`audio/capture.py:404`) |
| P3-1, -2, -4, -5, -7, -8, -9, -13 | Open (P3-34, -40, -31, -33, -12, -13, -36, -15) |
| P3-3 | Open (P2-8) |
| P3-6, P3-10 | Partly fixed (P3-14, P3-7) |
| Hardware audit: `--effect-duration` fails validation | Fixed (default is now a float) |
| UI audit U2, U8 | Fixed; U9-U12, U24 open (P3-30) |

Of the 24 core items in the 09-30 bug audit: 8 fixed, 5 partly fixed, 11 open.

---

## Design debt (one line each)

1. `App.run()` is one 1,900-line method; P1-1 is a direct product of
   loop-scoped locals behind a mutable flag.  `app.py` is 11,141 lines and
   `overlays.py` 7,516.
2. `hotkeys.handle()` is a 1,400-line if/elif chain; dispatching actions by
   replaying key chords is the root of P2-13 and P2-14.
3. Two guard mechanisms coexist: `CallGuard`, and bare try/except plus an
   unthrottled warning.
4. Every overlay primitive is its own array build, VBO write and draw call
   (175-423 draw calls for the HUD and system monitor).
5. Hit rectangles are rebuilt as a side effect of rendering, so input depends
   on the last drawn frame.
6. The three JSON stores duplicate their load/save/quarantine code; none
   fsyncs before the rename.
7. Drop-in discovery imports every top-level `.py` of every drop-in, three
   times over for effects, help and tour; a failing module is retried and
   re-logged each time.
8. PortAudio device indices are captured once at start; a device plugged in
   later never appears until restart.
9. No SIGTERM handler: a service stop skips recorder finalization and the
   state save.
10. `AudioProfile.beat_threshold`, `smoothing` and `curve` are set on all 27
    profiles and read by nothing; `BaseEffect.on_midi` and
    `capture._probe_device_openable` are dead code.
11. Stale docs in code: `unicornviz/__init__.py` lists eight effects that are
    no longer in core; `effects/base.py:67` says BPM is "not yet implemented";
    `--fps-limit` help says "default 30" (it is 0); the `_LogBandFilter`
    docstring says INFO suppresses warnings (it does not).

---

## What this audit did not do

- Did not launch the app, open a GL context, an audio stream or a MIDI port.
  Timings for overlays are Python-side lower bounds against a GL mock.
- Did not read the 27 profile tables line by line, and did not review any
  detector, director or recommender tuning value.
- Did not audit mouse-coordinate mapping under span/mirror, the config-editor
  row sanitization for malformed drop-in rows, or shader visual correctness.
- Did not test on Windows, on Python 3.11, or against Pillow 13.
- Did not fix anything.  P1-1, P2-1, P2-2, P2-17 and P3-15 are each a change
  of a few lines.
