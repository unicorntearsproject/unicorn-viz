# Webcam and Streaming audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/webcam-01` at `eee52af` (`1.9.0`): the camera overlay, capture
  worker, treatments, border and background segmentation (2,397 lines).
- `drop-ins/streaming-01` at `a135dcf` (`0.9.1`): the RTMP streamer and its
  encoder selection (894 lines).
- Core's side of both: loading, key dispatch, the webcam render step.

Method:

- Every line was read, including the shaders.  No sub-agent was used.
- **Streaming** was run end to end with the real local `ffmpeg`, writing to
  local FLV files instead of an RTMP server, and with two stand-in encoder
  scripts (one that never reads its input, one that exits at once with an
  error).  No real audio device was opened.
- Part-way through, the owner supplied a Rumble test endpoint and stream
  key that do not go live.  Four short video-only connections were made to
  it (two with the real key, one with a deliberately wrong key through the
  streamer, one wrong-key run of `ffmpeg` directly to read its error text).
  The key is not recorded in this report or in the repository.
- **Webcam** was run in an offscreen GL context with a stand-in for
  `cv2.VideoCapture`, so **no real camera was opened**.  Device-open counts
  and render-thread costs are real; the time one real device open takes was
  not measured.
- The app was not launched.  Timings are on this machine (16 CPU threads,
  **Mesa Intel Iris Xe**).
- **VERIFIED** = reproduced by script.  **By reading** = found in the code,
  not run.  Scripts are in `/var/tmp/uv-misc-audit/` (`t6.py` to `t9.py`).

Severity: **P1** = crash, data loss or runaway.  **P2** = wrong behavior the
audience or operator will notice, a routine frame-time hit, or a privacy or
secret-handling fault.  **P3** = minor or hygiene.

## Baseline

| Check | webcam-01 | streaming-01 |
|---|---|---|
| Own tests | 124 passed | 45 passed — **one test fails intermittently** |
| ruff | clean | clean |
| bandit (source) | 7 Low | 7 Low |

**Intermittent test failure, reported and not touched:**
`drop-ins/streaming-01/tests/test_writer_thread.py::test_stop_drains_writer_before_closing_stdin`
fails with `assert 0 == 2`.  It failed 1 time in 14 sequential runs and 5
times in 48 runs under parallel load.  By reading: the test asserts as soon
as `stop()` returns, and since 0.9.1 `stop()` hands teardown to a thread, so
the assertion races the writer.  The neighboring tests call
`wait_stopped()` first; this one does not.  The product behavior it checks
(queued frames are written before the pipe closes) held in the end-to-end
runs.

Bandit, reported and not remediated (all Low):

| ID | Where | Assessment |
|---|---|---|
| B105 "hardcoded password" ×4 | `webcam_overlay.py` | False positives: the strings are `'hidden'`, `'hide'` and `'fullscreen'` layout names |
| B110 ×3, B112 ×1 | `webcam_overlay.py` ×2, `rtmp_streamer.py` ×1; `webcam_overlay.py` ×1 | Deliberate best-effort teardown and parsing fallbacks |
| B404, B603 ×4, B607 | `rtmp_streamer.py` | `ffmpeg` and `pactl` are run with argument lists and no shell.  `pactl` is found by bare name.  Acceptable |

## Summary

No P1.

| ID | Sev | Drop-in | Finding |
|---|---|---|---|
| P2-1 | P2 | streaming | The default encoder ignores the bitrate setting: 16.7 Mbit/s measured at 720p30 — VERIFIED |
| P2-2 | P2 | streaming | `endpoint` and `stream_url` in the config are discarded unless `provider = "custom"` — VERIFIED |
| P2-3 | P2 | streaming | Why a stream failed is never shown; `start()` reports ON for a stream that dies a moment later — VERIFIED |
| P2-4 | P2 | streaming | A stalled encoder drops frames silently while the panel says LIVE; nothing reconnects after a failure — VERIFIED |
| P2-5 | P2 | streaming | Picture time follows the frame count, so it falls behind the sound whenever the app renders below the stream rate — VERIFIED |
| P2-6 | P2 | streaming | A stream key inside the custom endpoint is logged and shown in the Control Room; any key is visible in the process list — VERIFIED |
| P2-7 | P2 | webcam | Each new camera frame costs 13.8 ms on the render thread at 720p with default settings — VERIFIED |
| P2-8 | P2 | webcam | Turning the overlay off by MIDI leaves the camera capturing — VERIFIED |
| P2-9 | P2 | webcam | Camera discovery opens every video device on the render thread — VERIFIED (count) |

Plus 17 P3 items.

## Live test against the owner's Rumble test endpoint

1280×720 at 30 fps, video only, a moving noisy test picture (a hard case for
the encoder), frames offered at a steady 30 per second.

| Run | Result | Frames dropped at the queue | Upload measured | What the operator is shown |
|---|---|---|---|---|
| Real key, `libx264` (default) | Connected, stayed up 15 s | **249 of 450 (55%)** | 9.3 Mbit/s | LIVE, no error |
| Real key, `libx264`, 20 s | Connected, stayed up | **277 of 600 (46%)** | — | LIVE, no error |
| Real key, `h264_vaapi` (rate-limited to 1.5 Mbit/s) | Connected, stayed up 15 s | 16 of 450 (4%, at start-up) | 1.6 Mbit/s | LIVE, no error |
| Wrong key, `libx264` | `start()` returned success; down after 0.7 s | — | — | `stream exited with code 255` |

Run directly with its output visible, `ffmpeg` said for the wrong key:
`Server error: Invalid stream key`.

This confirms three findings on a real connection.  The default encoder
sends as much as the link will take (P2-1); here that was 9.3 Mbit/s, and
about half the frames never left the machine.  Nothing told the operator
(P2-4).  And a wrong key looks identical to any other failure (P2-3).  The
same picture through a rate-limited encoder went out almost whole.

## P2

### P2-1. The default encoder ignores the bitrate — VERIFIED

`_video_spec()` returns `-preset … -tune zerolatency` for `libx264` and
nothing else (`rtmp_streamer.py:108-109`).  The bitrate computed by
`_stream_bitrate()` (the `video_bitrate` setting, or the automatic
1.5–9 Mbit/s figure the docstring promises) is passed only to the other
encoders.  `libx264` is the default codec, and it then runs at its own
quality target with no ceiling.

```
video_bitrate = "4500k"  ->  -c:v libx264 -preset veryfast -tune zerolatency -pix_fmt yuv420p
```

A 6 s run at 1280×720, 30 fps, with a moving, noisy test picture measured
**16.7 Mbit/s**; the automatic figure for that size is 1.5.  Real visuals
vary, but busy full-screen effects are exactly what this encodes.  The
platforms this targets recommend single-digit Mbit/s for 1080p60, and an
upload link that cannot carry the stream turns into P2-4.

### P2-2. `endpoint` and `stream_url` are discarded — VERIFIED

The constructor reads `endpoint` and `stream_url` (`:308-313`) and then
calls `set_provider(self._provider, restart=False)` (`:336`).  For the
Rumble and YouTube providers that overwrites the endpoint with the provider
preset and blanks the override (`:560-569`).  The default provider is
Rumble.

```
endpoint = "rtmp://my.server/live", default provider  ->  rtmp://live-api-s.rumble.com/live
set in the config editor                              ->  rtmp://my.server/live
after Ctrl+F9 (Rumble)                                ->  rtmp://live-api-s.rumble.com/live
```

The shipped `config.toml` sets `endpoint` to the Rumble address, so the two
happen to agree.  The comment beside `stream_url` says "full override URL;
if set, endpoint/key are ignored"; in practice it is the override that is
ignored.  Only `provider = "custom"` with `custom_endpoint` reaches a
different server.

### P2-3. Failure reasons are hidden — VERIFIED

`ffmpeg` is started with its output sent to `DEVNULL` (`:664-665`), and
`is_streaming` is true as soon as the process exists (`:461-462`).

With a stand-in encoder that prints `Server error: rtmp: bad stream key`
and exits:

```
start() returned True          (operator sees "Streaming ON")
after a few frames: is_streaming False, last_error 'stream exited with code 1'
```

The operator learns that it stopped, not why: a wrong key, an unreachable
server and a missing audio device all read the same.

### P2-4. A stalled encoder drops frames silently — VERIFIED

`write_frame()` drops a frame when the queue is full and still returns
`True` (`:760-765`).  There is no drop counter, no "falling behind" state
and no timeout.  With a stand-in encoder that never reads its input, 600
frames were offered: the queue held 2, `is_streaming` stayed `True`,
`last_error` stayed empty, and the Control Room panel would show LIVE
indefinitely while nothing leaves the machine.

When a write does fail, the stream is stopped (`:754-756`) and stays
stopped; there is no reconnect or back-off, so a brief network drop ends the
broadcast until the operator presses F8 again.  By reading.

Stopping is sound: `stop()` returned in 0.4 ms against the stalled encoder
and the background teardown finished.

### P2-5. Picture time follows the frame count — VERIFIED

Frames go to `ffmpeg` as raw video at a declared rate (`:598-604`), so each
frame is stamped by its position in the stream, not by when it was drawn.
Audio is captured in real time.  If the app delivers fewer frames per second
than the stream's `fps`, the picture runs slow against the sound, and the
gap grows for as long as the stream runs.

With `fps = 30` and frames delivered at 20 per second for 8 s of real time,
the file held **5.3 s** of video.  At that ratio the picture would be a full
second behind the sound after three seconds on air.  The default stream rate
is 60, so any scene that renders below 60 fps while streaming drifts.

### P2-6. Stream key exposure — VERIFIED

- **Separate `stream_key`:** masked as `***` in the label, the log and the
  panel.  Good.
- **Key inside `custom_endpoint`** (the config editor's own hint for the
  endpoint row reads "RTMP URL (often with the stream key)"): the full URL,
  key included, is written to the log at INFO and shown in the Control
  Room's Destination row.  `destination_label` masks only the separate key
  (`:468-475`).
- **Process list:** in every case the key is on `ffmpeg`'s command line and
  readable by any local user with `ps`.  That is how `ffmpeg` takes its
  output URL; it is listed so the owner knows.

The key also has to live in the tracked `config.toml` or the runtime store
in plain text.

### P2-7. Webcam frame cost on the render thread — VERIFIED

`_get_camera_texture()` (`webcam_overlay.py:2287-2320`) runs on the render
thread for every new camera frame: a copy out of the worker, a horizontal
flip, optionally a floating-point brightness/contrast pass, a `tobytes()`
copy and a 3-channel upload.

| Camera | Defaults (flip H on) | Flip off | Brightness 1.2 |
|---|---|---|---|
| 1280×720 | **13.8 ms** | 4.4 ms | 27.0 ms |
| 1920×1080 | 21.4 ms | 9.9 ms | 60.2 ms |

At 30 camera frames a second, every other rendered frame pays this.  The
flip alone is about 9 ms at 720p, and it undoes a flip the capture thread
has just applied (`:664`): the worker mirrors every frame, and the default
`flip_horizontal = true` mirrors it back.  Any brightness or contrast other
than 1.0 doubles the cost again.  All three steps could run on the capture
thread or in the shader.

### P2-8. MIDI "toggle webcam" leaves the camera on — VERIFIED

`_midi_toggle()` flips `self.enabled` and nothing else
(`:1051-1053`).  The overlay stops drawing, but the capture thread keeps
running, so the camera stays open and its light stays on.  After the MIDI
toggle: `enabled False | capture thread still running: True`.

Hiding with Ctrl+KP `.` does stop capture (`:1203-1208`), and the drop-in
has a dedicated test file for that path.  The MIDI action is the gap.

### P2-9. Camera discovery opens every device on the render thread — VERIFIED (count)

`_enumerate_cameras()` opens each `/dev/video*` node in turn to see whether
it works (`:1465-1487`).  It runs from `start()`, from the first show after
being hidden, and from the Control Room's REDISC button, all on the calling
thread.

With 6 video nodes: 7 device opens at start (6 probes, then the worker's
own), and 6 more for a REDISC while a camera is live.  A USB camera
typically exposes two nodes, so three cameras is six.  Each open lights the
camera and takes real time; that time was not measured because no real
camera was opened here.  Probing also opens cameras the operator has
disabled in the list.

## P3

### streaming-01

| ID | Where | Finding |
|---|---|---|
| P3-1 | `rtmp_streamer.py:138-157`, `:654-656` | The first start probes encoders on the calling thread: 212 ms for a configured `libx264`, 406 ms for `auto` here — VERIFIED.  Each probe may wait up to 20 s on a broken driver, and up to five run in sequence |
| P3-2 | `:531` | `pactl info` is run with no timeout on the calling thread |
| P3-3 | `:64-65` | The probe cache is a module-level global and is never invalidated; replacing `ffmpeg` mid-session keeps the old answers |
| P3-4 | `:287`, `:326` | `fps` and `max_queued_frames` from config are not range-checked; a non-numeric value raises in the constructor and core falls back to no streamer |
| P3-5 | `:84-87` | The hardware encoder list for macOS is the Windows list (NVENC, QSV); VideoToolbox is not tried |
| P3-6 | `tests/test_writer_thread.py:113-127` | The intermittent test described under Baseline |
| P3-7 | docs, `config.toml:63` | The `stream_url` comment and the configuration doc describe an override that P2-2 discards |

### webcam-01

| ID | Where | Finding |
|---|---|---|
| P3-8 | `webcam_overlay.py:2227-2266` | The border and the transparent cutout rely on blending already being enabled by whoever drew before; the module never enables it.  With blending off, the futuristic border paints a solid box (48,500 pixels differ) — VERIFIED |
| P3-9 | `:1095-1100` | PageUp and PageDown are claimed with no modifier and always return a message, so they are consumed even with one camera or the overlay hidden |
| P3-10 | `:1084-1092` vs `:813-821` | Alt+`[` and Alt+`]` resize the PiP but are not in `HELP_ENTRIES`; the module docstring lists every key (project rule: `HELP_ENTRIES` only) |
| P3-11 | `:1853-1933` | Non-numeric values in the saved state (`pip_scale`, per-camera `brightness`) raise — VERIFIED |
| P3-12 | `:849-852`, `:890`, `:1372-1377` | Non-numeric `fps` or a malformed `border_color` raises in the constructor; a negative `cycle_interval` is accepted and makes auto-cycle flip every frame — VERIFIED |
| P3-13 | `:1216-1224`, `:2383-2385` | Stopping the worker joins it for up to 2 s on the render thread |
| P3-14 | `:609-639` | After five failed reconnect attempts (about 15 s) the worker gives up for good; a camera unplugged for longer needs a manual hide and show |
| P3-15 | `:664`, `:922`, `:2294-2298` | `flip_horizontal = true` shows the un-mirrored picture and `false` shows the mirrored one, because the capture thread always mirrors first |
| P3-16 | `:1263-1271`, `:1507-1522`, `:1984-2001` | The Control Room panel reads each camera's name from sysfs twice per panel frame |
| P3-17 | `:1348-1361`, `:1935-1938` | Every panel action rescans the camera identity folders and saves the whole runtime state |

## Recommended tests

Each is written to fail against the current code where a finding is open.
None needs a camera, a network or an audio device.

### streaming-01

| Test | Covers |
|---|---|
| `_build_command()` with `libx264` and `video_bitrate = "4500k"` contains a rate limit for the video encoder | P2-1 |
| A streamer built with `endpoint` or `stream_url` and the default provider resolves to that URL | P2-2 |
| With an encoder that exits non-zero after writing to stderr, `last_error` contains that text, and `start()` does not report success | P2-3 |
| With an encoder that never reads, a dropped-frame count rises and a "stalled" state is reported within a stated time | P2-4 |
| After a write failure the streamer retries with a back-off instead of staying stopped | P2-4 |
| Frames delivered at two-thirds of the stream rate produce a file whose video duration matches wall time (frames duplicated or timestamps from the clock) | P2-5 |
| No log line and no panel row contains a key embedded in `custom_endpoint` | P2-6 |
| The first `start()` returns without running an encoder probe on the calling thread | P3-1 |
| `test_stop_drains_writer_before_closing_stdin` waits for teardown before asserting | P3-6 |
| Non-numeric `fps` falls back to the default | P3-4 |

### webcam-01

| Test | Covers |
|---|---|
| With a fake worker delivering 720p frames, `_get_camera_texture()` performs no flip and no float pass on the render thread at default settings | P2-7 |
| After `_midi_toggle()` turns the overlay off, no capture thread is alive; turning it back on restarts capture | P2-8 |
| `start()` and `rediscover_cameras()` with six video nodes open no device on the calling thread | P2-9 |
| Cameras marked disabled are never opened by discovery | P2-9 |
| `render()` leaves the picture identical whether blending was on or off beforehand | P3-8 |
| PageUp with a single camera returns `False` | P3-9 |
| Every key `handle_key()` acts on appears in `HELP_ENTRIES` | P3-10 |
| Saved state and config with non-numeric values fall back to defaults; a negative `cycle_interval` is clamped | P3-11, P3-12 |

## What works

- **Stream teardown.**  `stop()` never blocks the caller (0.4 ms against a
  stalled encoder), escalates to a kill in bounded steps, and a provider
  switch while live restarts cleanly.
- **Encoder fallback.**  The configured codec is tested with the stream's
  own flags before use, hardware encoders are tried first on `auto`, and a
  machine with no H.264 encoder gets a message that says how to fix it.
- **Frame hand-off.**  All pipe writes happen on a writer thread; the render
  thread only queues.  A 6 s local encode kept 175 of 180 frames.
- **Separate stream key.**  Masked everywhere it is displayed or logged.
- **Webcam privacy when hidden.**  The camera is not opened while the
  overlay starts hidden, and hiding releases it; this is covered by tests.
- **Webcam frame reuse.**  Nothing is uploaded when the camera has not
  produced a new frame, and capture and segmentation run off the render
  thread.
- **Camera identity.**  Per-camera settings follow a camera across USB
  renumbering and are parked while it is unplugged.
- **Teardown.**  The webcam releases every GL object it creates.

## What this audit did not do

- Open a real camera or capture real audio.  Camera-open time and A/V sync
  against a live audio device were not measured.  The live streaming runs
  were video only and used a synthetic picture, not the app's own output.
- Run background segmentation; `mediapipe` behavior was read, not run.
- Launch the app or drive hotkeys, MIDI or the Control Room through real
  input.
- Test Windows or macOS paths (`dshow`, `CTRL_BREAK_EVENT`, NVENC, QSV).
- Fix anything, including the intermittent test.  The bandit findings are
  reported only.
