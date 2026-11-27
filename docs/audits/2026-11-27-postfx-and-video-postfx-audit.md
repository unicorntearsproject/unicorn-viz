# Post-FX and video post-FX audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/postfx-01` at `3870e8e` (`1.0.0-rc.2`): the controller and its 12
  effect files (2,177 lines).
- `drop-ins/video-postfx-01` at `972980d` (`0.3.0`): controller, chroma
  helpers and config validator (693 lines).
- Core's side of both: the post chain in `unicornviz/app.py` and the post-FX
  functions in `unicornviz/vj_api.py`.

Method:

- Every line of both drop-ins was read, including every shader.  No
  sub-agent was used.
- Both were run in an offscreen GL context: each post-FX slot through its
  full hit, the three-pass chain, test patterns for the vignette and
  rotation, cost at 1080p and 4K, and create/resize/destroy cycles with a
  GL-object count.
- The video overlay was run in real time against small clips generated into a
  scratch folder with PyAV (a looping clip, an unreadable file, an oversize
  clip, a clip deleted after the scan).
- No window was opened and the app was not launched.  Timings are on **Mesa
  Intel Iris Xe**, the only GPU on this machine.
- **VERIFIED** = reproduced by script.

Scripts and clips: `/var/tmp/uv-pfx-audit/`.

Severity: **P1** can crash or freeze the app or stall the audience output.
**P2** visibly wrong on the audience output, a log flood, or a measurable
cost on the render thread.  **P3** latent, cosmetic or a rule violation.

---

## Baseline

| Check | Result |
|---|---|
| postfx-01 tests | 7 passed (all seven cover the Control Room panel; nothing tests `apply()` or any effect) |
| video-postfx-01 tests | 29 passed |
| Shaders | all compile; none raises on NaN or infinite audio values |
| GL leaks | none: 134 live objects before and after 20 create / resize / destroy cycles |
| Version sync | both drop-ins: code, README header and changelog agree |
| Docs | both have the four required docs |

---

## Summary

| ID | Sev | Where | One line |
|---|---|---|---|
| P2-1 | P2 | postfx | With a slot, hue shift and rotation all active, the hue pass reads the texture it is writing — **VERIFIED** |
| P2-2 | P2 | postfx | Nine of the ten slots end with a hard cut instead of fading out — **VERIFIED** |
| P2-3 | P2 | postfx | The "vignette" darkens the center more than the edges — **VERIFIED** |
| P2-4 | P2 | video-postfx | A looping clip disappears for one frame at every loop — **VERIFIED** |
| P2-5 | P2 | video-postfx | A clip that cannot be opened starts a new thread and logs a traceback on every frame, until cleared — **VERIFIED** |
| P2-6 | P2 | video-postfx | Each new 1080p video frame costs 8 ms on the render thread, per layer — **VERIFIED** on this GPU |
| P3-1…P3-18 | P3 | both, core | See the P3 tables |

No P1 was found.  One prior core finding applies directly: the post chain is
called unguarded, so an exception from either controller's `apply()` ends the
show (core audit P1-2).  Nothing in this pass made either one raise.

---

## P2

### P2-1. The three-pass chain feeds back on itself — VERIFIED

`postfx_controller.py:490-522`.  The controller has one internal buffer
(`_hue_fbo`) for chaining.

- Slot + hue, slot + rotation, hue + rotation: correct.  Each pass reads one
  buffer and writes another.
- **Slot + hue + rotation:** the slot writes `_hue_fbo`; the hue pass then
  reads `_hue_fbo`'s texture and writes `_hue_fbo` (line 501-502, because
  rotation is also running); rotation reads it.  Sampling a texture while
  rendering into it is undefined in OpenGL.

Measured: the hue pass's source is its own destination's attachment, and the
output differs from the correct three-buffer chain by a mean of 128 of 255
per channel.

It needs all three at once: a slot hit (Auto VJ fires these) inside the
3-second idle windows of both a wheel hue shift and a Ctrl+wheel rotation.

**Fix:** a second internal buffer, ping-ponged.

### P2-2. Slot hits end with a cut — VERIFIED

`DEVELOPER_NOTES.md` states the contract: "Strength envelope: 1.0 at trigger,
fades to 0.0 at expiration."  The controller does ramp `strength` to 0
(`postfx_controller.py:491-495`), but nine of the ten effects add a floor that
does not scale with it, so they are far from the source image on their last
frame and then switch off.

Mean difference from the source image, 0-255, silent audio:

| Slot | Effect | Just after trigger | Last frame of the hit |
|---|---|---|---|
| 6 | Multi-pass Bloom | 126 | 81 |
| 8 | Temporal Feedback Trail | 6 | 54 |
| 5 | Lens Distortion + Vignette | 93 | 45 |
| 9 | Time Scramble Warp | 52 | 29 |
| 7 | Radial Zoom Blur | 46 | 16 |
| 4 | Heat Haze Refraction | 43 | 16 |
| 2 | Film Grain + Dither | 17 | 9 |
| 3 | Glitch Slices | 31 | 9 |
| 1 | Chromatic Aberration | 21 | 6 |
| 10 | Smoke & Bubbles | 14 | 0 |

Only Smoke & Bubbles reaches the source image (`mix(src, col, s)`).  The
others carry constants such as `0.28 + 0.62 * s` (Time Scramble), an
unconditional tone map (Bloom, `multi_pass_bloom.py:103-104`) or a fixed
contrast curve.  Hue shift and rotation fade correctly.

Whether a hit should punch out or fade is a taste call; the code's own notes
say fade.

**Fix:** blend each effect's result with the source by `strength` at the end,
as Smoke & Bubbles does.

### P2-3. The vignette is inverted — VERIFIED

`effects/lens_distortion_vignette.py:40-42`.

```
float vig = smoothstep(0.12, 0.95, dot(uv, uv));
c *= mix(1.0, vig, uVignette);
```

`vig` is 0 at the center and rises toward the corners, and the color is
multiplied by it.  On a white frame at full strength: center 65 of 255,
mid-edge 76, corner 144.  The comment says "strong at edges, fade to center";
the picture is a dark center with lighter corners, and the whole frame is
dimmed.  `uVignette` also has no upper clamp (`0.75 × … + bass × 0.15`, with
raw `bass`).

**Fix:** `c *= mix(1.0, 1.0 - vig, clamp(uVignette, 0.0, 1.0))`.

### P2-4. Video layers blink at every loop — VERIFIED

`video_postfx_controller.py:475-479`.

```
eof = layer.pump(dt)
if eof:
    layer.restart()  # always loop
    return False
```

On the frame that reaches end-of-file the layer is not drawn, although its
texture still holds the last frame.  A one-second clip at 60 fps over five
seconds: the overlay was missing on frames 57, 115, 173, 231 and 289.  Each
loop also opens the file again and starts a new decoder thread.

**Fix:** restart and fall through to the draw, so the last frame holds until
the new decoder delivers.

### P2-5. An unreadable clip floods threads and the log — VERIFIED

`video_postfx_controller.py:110-123`, `177-199`, `475-479`.  A decoder that
fails to open puts an end-of-stream marker on its queue.  The controller
treats that as "end of clip" and restarts the layer, which fails again.

- **Garbage file with a video extension:** in 2.0 s (115 frames), 114 decoder
  threads were started and 115 warnings logged, each with a full traceback
  (`exc_info=True`).  The layer keeps its slot.
- **Clip deleted after the folder was scanned:** 58 warnings in one second.
  The pool is scanned once and never refreshed while it is non-empty
  (`370-384`), so a clip removed or renamed during a session triggers this
  the next time the shuffle picks it.
- **A file with no video stream** (by reading, `135-138`): the same restart
  loop with no log line at all.

It continues until the operator clears the layers.

**Fix:** count consecutive failed opens per layer; drop the layer after two
or three, with one warning and a flash message; remove the path from the pool.

### P2-6. Video frame upload cost — VERIFIED on this GPU

`video_postfx_controller.py:491`.  Each new decoded frame is copied
(`arr.tobytes()`) and uploaded synchronously on the render thread.  For one
1920×1080 RGB frame: 8.4 ms mean, 12.5 ms worst.  That is per layer, per new
video frame; up to four layers by default, eight selectable.  Four 1080p
30 fps layers would spend about 33 ms on every other render frame here.

Uploading the array without the copy measured no better (9.9 ms), so the cost
is the upload itself on this integrated GPU; a discrete GPU was not available
to compare.  The clips in the tests were 320×180, where this is negligible.

**Fix:** downscale on the decoder thread to a capped size (the music-video
decks do this with `cache_long_edge = 720`); or upload through a pixel
buffer.

---

## P3

### postfx-01

| ID | Where | What |
|---|---|---|
| P3-1 | `effects/scene_rotation.py:25-29` | Rotation is done in UV space with no aspect correction: at 90° on 16:9 a circle 270 px across becomes 478 × 151 — **VERIFIED** |
| P3-2 | `postfx_controller.py:372-388`, `397-402`, `519-522` | The rotation angle and hue offset are kept after the idle timeout: one more wheel notch jumps straight back to the old accumulated value (1.571 rad kept, next notch 1.641) — **VERIFIED** |
| P3-3 | `effects/glitch_slices.py:93`, `film_grain_dither.py:116`, `time_scramble_warp.py:130` | These three reset their clock to 0 on each trigger, so every hit plays the identical pattern (frame difference 0.000 between triggers) — **VERIFIED** |
| P3-4 | seven effect files | `log.debug(...)` on every frame inside `apply()` (bloom, grain, glitch, heat haze, lens, radial blur, time scramble): a log line per frame at DEBUG level |
| P3-5 | effect `apply()` methods | Raw `bass` / `mid` / `treble` (which can exceed 1) scale uniforms without a clamp: bloom intensity, vignette, lens distortion, heat-haze amount |
| P3-6 | `postfx_controller.py:176-208`, `215` | The `float(cfg.get(k, d) or d)` idiom makes 0 unsettable for every duration and timeout; a non-numeric value raises in the constructor |
| P3-7 | `postfx_controller.py:296-319` | The slot hotkeys use the Ctrl+Alt leader.  `video-postfx-01`'s own docs call Ctrl+Alt unreliable because desktop compositors grab it; on Windows AltGr is delivered as Ctrl+Alt, so AltGr+digit fires a slot |
| P3-8 | `postfx_controller.py:440-445`; `temporal_feedback.py:139-145`; `multi_pass_bloom.py:226-234` | `trigger_slot()` calls `reset()`, which binds an internal framebuffer and clears it, from wherever the trigger came (hotkey, Auto VJ update, Control Room action): GL state changes outside the render pass.  Harmless today because every pass rebinds its target |
| P3-9 | all effect shaders | GLSL conventions: uniforms use a `u` prefix where the project convention is `i`; 15 of 17 shader strings have no stage/uniform comment block; `smoke_bubbles.py`'s fragment shader is 176 lines |
| P3-10 | `tests/` | No test exercises `apply()`, the chain, or any effect; P2-1 to P2-3 would each be caught by a small offscreen test |

### video-postfx-01

| ID | Where | What |
|---|---|---|
| P3-11 | `video_postfx_controller.py:482-484`, `82` | A clip wider or taller than 2,048 px is skipped on every frame with no message: the layer holds a slot and never appears — **VERIFIED** with a 2560×1440 clip |
| P3-12 | `video_postfx_controller.py:101-108` | `_Decoder` stores its stop flag as `self._stop`, which shadows a private `threading.Thread` method.  On Python 3.11 and 3.12, `is_alive()` and `join()` on a decoder raise `TypeError: 'Event' object is not callable` — **VERIFIED**; fine on 3.14.  Latent: the drop-in never calls either |
| P3-13 | `video_postfx_controller.py:204-208`, `537-549` | Decoder threads are stopped but never joined, on close and on destroy; they are daemon threads that may still be inside the decoder when the interpreter exits |
| P3-14 | `video_postfx_controller.py:85-89`; `.gitignore` | The default clip folder is `videos/` inside the drop-in, and `.gitignore` does not ignore its contents: the owner's clips show up as untracked files in the submodule.  A `video_dir` override is resolved against the working directory, not the app root, and is not confined to the project |
| P3-15 | `chroma.py:32-50`; controller `492-494` | The key color is taken once, from the first frame delivered.  A clip that opens on anything other than its flat background keys the wrong color for its whole run; the documented `key_color` override is global, not per clip |
| P3-16 | `video_postfx_controller.py:307`; `config_validator.py` | `max_layers` is clamped to 1-16 only by the validator; the config-menu row offers 1, 2, 4, 8.  `key_tolerance` as a string raises in the constructor |

### Core side

| ID | Where | What |
|---|---|---|
| P3-17 | `unicornviz/vj_api.py:2198-2211` | `set_postfx_slot()` decides success by searching the returned message for fixed strings.  The controller's text is "use Ctrl+Alt+1..10"; core searches for "use ctrl+alt+1..9", so a negative slot reports success.  `hold_postfx_slot()` ignores its duration |
| P3-18 | `unicornviz/vj_api.py:2394-2409`, `2252` | `set_hue_offset()` and `set_rotation_degrees()` write the controller's private fields directly; `postfx_slot_duration()` reads `_slot_hit_duration`.  The controller has public methods for the first two |

---

## What works

- **Robust inputs.**  NaN and infinite audio values raise in no slot, and the
  8-bit targets cannot hold a non-finite pixel.
- **No leaks, clean resize.**  Every effect releases what it creates; resize
  rebuilds the internal buffers.
- **Cost is low.**  Every slot is 1.4-3.7 ms at 1080p on this integrated GPU.
  The heaviest at 4K is Smoke & Bubbles at 12.8 ms.
- **Smoke & Bubbles survives a long night.**  Its clock only moves forward,
  but its hash bounds its input: frame-to-frame motion and bubble counts are
  the same at 10 s and at 50,000 s.
- **The two-pass chains are correct**, and the controller's threading is
  simple: everything runs on the main thread; the Control Room panel only
  reads properties.
- **Video overlay basics hold.**  The shuffle bag never repeats a clip until
  all have played; decoding is off the render thread with a bounded queue;
  replacing the oldest layer releases its texture; `destroy()` is idempotent
  and is reached from core's shutdown list.
- **Key handling is scoped.**  Both drop-ins list every key they handle in
  `HELP_ENTRIES`, and neither reaches into `app` private state.

---

## What this audit did not do

- No on-screen run, no real show footage, no discrete GPU.
- Did not judge how any effect looks beyond what could be measured.
- Did not exercise a file with no video stream (P2-5, third case) or decoder
  behavior at interpreter exit (P3-13).
- `beat-flash-01`, `color-grade-01` and `candy-frame-01` share the post chain
  and were not part of this request.
- Did not fix anything.  P2-3 and P2-4 are one line each; P2-1 is one extra
  buffer.
