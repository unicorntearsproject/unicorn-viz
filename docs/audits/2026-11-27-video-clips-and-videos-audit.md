# Video Clips and Videos audit — performance and clip loading (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/video-clips-01` at `1a7d147` (`0.5.0`): the Video Clips montage
  effect (460 lines).
- `drop-ins/videos-01` at `5536848` (`0.8.2`): the Video Player effect
  (528 lines) and `DeckVideoSource`, the frame cache behind music-video
  decks (497 lines).
- Core's consumer of the deck source, `unicornviz/video_deck_layer.py`, for
  the upload path only.

The owner asked for a special focus on performance and clip loading, so this
report is measurement-first and ends with a ranked list of measured
optimization options.

Method:

- Every line of both drop-ins was read.  No sub-agent was used.
- Test clips were generated into `/var/tmp/uv-video-audit/clips/` with PyAV:
  H.264 at 720p24, 1080p30, 1080p60 and 2160p30, with a 2 s keyframe
  interval, plus a 1080p30 file with a 10 s keyframe interval and one with
  no audio track.
- The real clip folders were inspected for metadata only (count, resolution,
  codec, frame rate): 6 clips, H.264, five at 1280×720 and one at 960×960,
  24 or 30 fps, about 10 s each, 114 MB.  Both drop-ins hold the same six.
  No owner clip was decoded or copied.  The synthetic 720p clips are about
  7 Mbit/s; the real ones are about 14, so real decode times will be
  somewhat higher than the figures here.
- Video Clips and the Video Player were run in an offscreen GL context at a
  1920×1080 target.  The Video Player's audio device was simulated (a thread
  calling its audio callback at a fixed block size); no real audio stream
  was opened.  `DeckVideoSource` was driven at 60 Hz the way core drives it.
- The app was not launched.  All timings are on **Mesa Intel Iris Xe** with
  16 CPU threads, the only GPU on this machine, with no other load.
- **VERIFIED** = reproduced by script.  **By reading** = found in the code,
  not run.  Scripts are in `/var/tmp/uv-video-audit/`.

Severity: **P1** = the picture freezes or goes badly stale, a routine stall
of 100 ms or more, or runaway CPU.  **P2** = visible judder, a visible pop,
unbounded growth, or behavior that contradicts the docs.  **P3** = minor or
hygiene.

## Baseline

| Check | video-clips-01 | videos-01 |
|---|---|---|
| Own tests | none in the drop-in (core has `tests/test_video_clips_groups.py`) | 21 passed |
| ruff | clean | clean |
| bandit (source) | clean | 1 Low |

Bandit, reported and not remediated: `B110` try/except/pass at
`deck_video_source.py:289`, Low.  Deliberate: closing the container during
worker shutdown.

## Summary

| ID | Sev | Where | Finding |
|---|---|---|---|
| P1-1 | P1 | deck source | On long-keyframe-interval files a cue jump never reaches its target and the worker spins about five CPU cores — VERIFIED |
| P1-2 | P1 | deck source | Every cue jump leaves the picture stale for 0.4 to 2.5 s — VERIFIED |
| P1-3 | P1 | Video Clips | Activation stalls the frame about 300 ms and every clip change about 150–200 ms (720p) — VERIFIED |
| P2-1 | P2 | Video Clips | Decode and upload of two clips run on the render thread; 13% of frames miss 16.7 ms at 720p, 49% at 1080p — VERIFIED |
| P2-2 | P2 | Video Clips | Every visited clip stays open: about 66 MB and 31 OS threads each — VERIFIED |
| P2-3 | P2 | Video Clips | The picture jumps sideways at every clip change — VERIFIED |
| P2-4 | P2 | Video Clips | Playback speed depends on the render frame rate — VERIFIED |
| P2-5 | P2 | Video Player | With no audio device, a video that has sound freezes after 4 frames — VERIFIED |
| P2-6 | P2 | Video Player | Frame pacing follows the audio block size; 22% of frames dropped at 43 ms blocks — VERIFIED (simulated device) |
| P2-7 | P2 | Video Player | Full-resolution frames are copied and uploaded on the render thread; 4K drops 21% of frames — VERIFIED |
| P2-8 | P2 | deck source | Reverse play is mostly stale: 74% fresh at 720p, 56% at 1080p, 2% on a long-GOP file — VERIFIED |
| P2-9 | P2 | deck source | The downscale uses the slow scaler; forward play of one 1080p deck costs 1.2 cores — VERIFIED |

Plus 14 P3 items.

## P1

### P1-1. A cue jump on a long-GOP file never arrives — VERIFIED

`_seek_and_refill()` (`deck_video_source.py:414-457`) seeks to the keyframe
at or before `target − 2 s` and converts and keeps **every** frame from
there on.  When the kept list reaches the memory cap (`:452`) it publishes
and returns, even if it has not reached the target yet.  The trim then keeps
almost nothing, the next loop sees the target far past the ring, and seeks
again to the same keyframe.

1080p30 file with a 10 s keyframe interval, core's default 720 px long edge
(cap 179 frames = 6.0 s):

| Jump to | Result after 5 s | Seeks | Process CPU |
|---|---|---|---|
| 13.0 s (3 s after a keyframe) | fresh frame after 0.94 s | 1 | 155% |
| 19.5 s (9.5 s after a keyframe) | **never**; nearest cached frame is 3.57 s away | 3 and counting | **475%** |
| 15.0 s at 1280 px long edge | **never**; 3.17 s away | 8 and counting | **522%** |

The jump fails whenever the target is more than about `cap + 1.5 s` after
its keyframe.  That is 7.5 s at the default setting for 30 fps video, 4.5 s
for 60 fps video (the cap is counted in frames), and 4.9 s at a 960 px long
edge.  x264's default keyframe interval is 250 frames (8–10 s), so ordinary
downloaded music videos are in range.  While the deck keeps playing, the
picture stays stale until the playhead comes within reach of the next
keyframe, and the worker burns several cores the whole time.

### P1-2. Every cue jump leaves the picture stale — VERIFIED

Same code path, working as designed.  Seeking 2 s before the target means at
least 2 s of video (plus the distance back to the keyframe) is decoded **and
downscaled** before the first usable frame is published (`:448-451`).  The
downscale is the expensive part (P2-9).

Time from a jump until a frame within one frame interval of the target is
available:

| File | Current | Seek to the target itself, convert only the target frame |
|---|---|---|
| 720p24, 2 s GOP | 430–770 ms | 35–80 ms |
| 1080p30, 2 s GOP | 800–1,470 ms | 58–181 ms |
| 1080p30, 10 s GOP | 1,050–1,840 ms, or never (P1-1) | 253–653 ms |
| 1080p60, 2 s GOP | 2,140–3,630 ms | 66–267 ms |
| 2160p30, 2 s GOP | 1,900–2,500 ms | 262–660 ms |

The right-hand column is a measurement of what the alternative costs with
the same library on the same files, not a patch.  The 0.8.2 changelog lists
cue-jump latency as fixed; the partial publish it added helps only after the
2 s run-up has already been paid.

Opening a file is fine: ready in 16–81 ms and first frame 48–231 ms later,
all off the render thread.

### P1-3. Video Clips stalls on activation and at every clip change — VERIFIED

All file opens and seeks happen on the render thread.

- **Activation.**  `_init` probes every clip in the chosen group by opening
  it (`video_clips.py:275-277`, `:302-310`): 165 ms for 6 clips and 942 ms
  for 60 on first use (cached afterwards).  The first `update()` then opens
  two clips and decodes a frame each (`:331-336`): 128 ms.  The first
  `render()` seeks both to a random position, because playback position is
  derived from the randomized effect clock (`:397-403`): 168 ms.  Together
  about **300 ms** at 720p after the probe, about **1.6 s** at 4K.
- **Every clip change.**  The next clip is opened in `update()` (47–101 ms)
  and seeked in the following `render()` (65–120 ms): a **150–200 ms hitch**
  every `mix_time`.

Per-file costs:

| File | Probe | Open + first frame | Seek + read |
|---|---|---|---|
| 720p | 14 ms | 59 ms | 93 ms |
| 1080p | 31 ms | 66 ms | 120 ms |
| 2160p | 95 ms | 294 ms | 392 ms |

## P2

### P2-1. Video Clips decodes and uploads on the render thread — VERIFIED

`render()` calls `_clip_frame_texture()` for both clips (`:421-422`), each
doing decode, BGR→RGB conversion, a vertical flip, a `tobytes()` copy and a
texture upload (`:404-412`).  Both clips are always decoded, since the
crossfade runs for the whole `mix_time`.

`render()` time at 60 fps, 1080p target:

| Group | Mean | p50 | p95 | Frames over 16.7 ms |
|---|---|---|---|---|
| 720p24 | 7.4 ms | 1.7 ms | 20.8 ms | 13% |
| 1080p30 | 14.4 ms | 15.8 ms | 37.1 ms | 49% |
| 1080p60 | 24.5 ms | 22.5 ms | 35.2 ms | 95% |
| 2160p30 | 38.2 ms | 29.8 ms | 87.6 ms | 50% |

The low median and high p95 at 720p are because both clips share a frame
rate, so both decode on the same render frames: 24 heavy frames a second.

Where one clip's frame time goes (median, ms):

| Step | 720p | 1080p | 2160p |
|---|---|---|---|
| Decode (`cap.read`) | 1.04 | 1.73 | 4.50 |
| BGR→RGB | 0.25 | 0.39 | 1.63 |
| Flip | 0.52 | 1.07 | 4.07 |
| `tobytes()` copy | 0.52 | 0.99 | 2.43 |
| Texture upload (3-channel) | 4.89 | 6.93 | 19.91 |
| **Total** | **7.22** | **11.12** | **32.55** |

The upload is the largest single item at every size.  See "Optimization
options" for what the alternatives measure.

### P2-2. Every visited clip stays open — VERIFIED

`_ensure_loaded()` (`:328-349`) opens a capture and a texture for each clip
as it comes up, and nothing closes them until `destroy()` (`:448-457`).  The
module docstring says "memory stays bounded".

After cycling through a 60-clip 720p group: 60 open captures, **+3.2 GB
resident memory**, **+1,860 OS threads** (31 per capture, from the decoder's
thread pool on a 16-thread CPU), 60 file handles and 166 MB of textures.
One capture measured 66 MB.  After `destroy()` the threads and handles are
gone but the resident memory is not returned to the system (it plateaus
rather than leaking: repeated open/close cycles of 6 clips settled at
+220 MB).

With today's 6-clip library this is about 400 MB and 186 threads while the
effect is on screen.

### P2-3. The picture jumps at every clip change — VERIFIED

The two shader slots use different pan phases and zoom terms
(`video_clips.py:207-211`: drift 0.1 vs 1.4, bass vs treble zoom).  When the
incoming clip finishes fading in, it moves from slot B to slot A and
instantly takes slot A's pan and zoom.  The style index changes on the same
frame, which also steps the vignette strength and tint.

With the same still image loaded in every slot, the frame-to-frame change is
normally 1.3–1.7 (mean absolute, out of 255); on the clip-change frame it
was 18–27, with a measured picture shift of up to 72 px horizontally and
40 px vertically at 1080p, in six of six trials.

### P2-4. Playback speed depends on the render frame rate — VERIFIED

The target frame index comes from the clock, but only one frame is read per
render unless the index is more than 3 ahead (`:401-404`).  A 60 fps clip
rendered for 3 s advanced 3.00 s of video at 60 fps, 1.50 s at 30 fps and
1.00 s at 20 fps.  Any time the app drops below the clip's frame rate the
clip runs in slow motion.  Past a gap of 3 frames it seeks instead, and a
seek costs 90–390 ms (P1-3), which lowers the frame rate further.

### P2-5. No audio device: a video with sound freezes — VERIFIED

When the output stream cannot be opened, `update()` runs the picture from a
wall clock (`video_player.py:459-464`).  But nothing reads the audio queue
in that case, so it fills (32 chunks), the decoder blocks on it
(`:204-210`), and no more video frames arrive.

Simulated with a failing device, 5 s run: a file with audio showed **4
frames** and stopped at 0.17 s with the audio queue at 32/32; the same test
with a file that has no audio track played normally (148 frames to 4.97 s).
The 0.7.1 changelog lists this case as fixed.  The regression test for it
uses a stub decoder with no audio queue, so it checks only that the clock
advances.

### P2-6. Frame pacing follows the audio block size — VERIFIED (simulated)

The video clock is the count of audio samples handed to the device, which
only moves when the audio callback runs (`:422`, `:458`).  Between callbacks
the clock stands still, then jumps by a whole block.

1080p30 video, 60 fps render, simulated device:

| Audio block | Frames shown | Time between new frames (ideal 33 ms) |
|---|---|---|
| 512 (11 ms) | 179 of 180 | p50 34, max 56 ms |
| 1,024 (21 ms) | 237 of 238 | — |
| 2,048 (43 ms) | 140 of 179 (22% dropped) | p50 47, max 64 ms |
| 4,096 (85 ms) | 71 of 180 (61% dropped) | p50 84, max 103 ms |
| 8,192 (171 ms) | 39 of 180 (78% dropped) | p50 166, max 189 ms |

The stream is opened with default settings (`:295-300`), so the block size
is whatever the device picks.  This machine's default output reports a
35 ms default latency, which falls between the second and third rows.  The
real block size was not measured because no real stream was opened.

### P2-7. Video Player uploads full-size frames on the render thread — VERIFIED

The decoder converts each frame to RGB at source resolution
(`:188`), and `render()` copies it with `tobytes()` and uploads it
(`:494`).  Nothing scales to the output size.

| File | Render on upload frames | Frames dropped | First picture |
|---|---|---|---|
| 720p24 | 4.4 ms (p95 7.9) | 0% | 57 ms |
| 1080p30 | 8.2 ms (p95 15.3) | 0% | 121 ms |
| 2160p30 | p95 44.8, max 87 ms | 21% | 351 ms |

The queue holds 4 decoded frames plus up to 2 in hand; at 4K that is about
150 MB.  Decoder startup, reopening and teardown are clean: 15 rapid reopens
of a 4K file left 2 extra threads, none after `destroy()`.

### P2-8. Reverse play is mostly stale — VERIFIED

The ring covers 2 s either side of the playhead.  Playing backward walks off
its trailing edge, which triggers a full seek-and-refill (the P1-2 cost)
every 2 s of travel.

| File | Rate | Fresh frames | Frame error p95 | Distinct frames shown |
|---|---|---|---|---|
| 720p24 | −1× | 74% | 667 ms | 107 of 144 |
| 720p24 | −2× | 36% | 1,468 ms | 102 of 288 |
| 1080p30 | −1× | 56% | 1,200 ms | 102 of 180 |
| 1080p30 | −2× | 17% | 4,302 ms | 62 of 360 |
| 1080p30, 10 s GOP | −1× | 2% | 3,751 ms | 3 of 180 |

Short scratches are fine: a ±0.4 s wobble at 2 Hz was 100% fresh with no
seeks.  The module docstring describes reverse play as "pure cache reads";
that holds only inside the 2 s already cached.

### P2-9. The downscale uses the slow scaler — VERIFIED

`_downscale()` (`:391-393`) calls `frame.reformat()` with the default
interpolation.  Per 1080p frame to 720×405:

| Interpolation | ms per frame |
|---|---|
| Default (bilinear) | 10.30 |
| Fast bilinear | 3.81 |
| Point | 3.64 |
| Fast bilinear to YUV, no RGB step | 3.35 |

Decoding the same frame takes about 0.6 ms, so the scaler is over 90% of the
worker's time.  Process CPU during forward play of one deck: 59% of a core
at 720p, 123% at 1080p, 363% at 4K.  Forward play itself keeps up (100%
fresh at 720p and 1080p at 1× and 1.5×; 95–99% at 4K).

## P3

| ID | Where | Finding |
|---|---|---|
| P3-1 | `video_clips.py:363-364` | A beat adds 0.12 to the crossfade for exactly one frame, then it snaps back (0.125 → 0.249 → 0.133) — VERIFIED.  Reads as a one-frame flicker on every beat rather than a push |
| P3-2 | `video_clips.py:428-445`, `video_player.py:502-511` | `render()` defines a closure and builds tuples every frame, and swallows a missing uniform silently.  Project rule: no allocation in the render hot path |
| P3-3 | `video_clips.py:54`, `:296-299` | `_META_CACHE` is a module-level global keyed by path and never invalidated; a replaced file keeps its old frame count and size.  Project rule: no global state |
| P3-4 | `video_clips.py:397-399` | `speed` multiplies the absolute clock, so any change to it makes both clips jump to an unrelated position (and seek).  By reading |
| P3-5 | both `scan_groups()` | The folder is rescanned recursively on the render thread at every activation; `video_dir` may point anywhere on disk.  The function is duplicated in the two drop-ins |
| P3-6 | `video-clips-01/` | No tests in the drop-in; no test covers decode, loading or teardown |
| P3-7 | `video_player.py:422`, `:458` | Picture leads sound by the decoder's startup time, because the clock starts before the first audio arrives: 0 ms at 720p, 21 ms at 1080p, 107 ms at 4K — VERIFIED.  Any later audio underrun adds to it permanently.  Device output latency is not subtracted either (by reading) |
| P3-8 | `video_player.py:293-304`, `:516-522` | The audio stream is opened in `_init` and stopped in `destroy()` on the render thread, and is opened even when there are no videos.  By reading; not timed, since no real stream was opened |
| P3-9 | `video_player.py:186-189` | Rotation metadata and non-square pixels are ignored.  By reading |
| P3-10 | `deck_video_source.py:233-242` | `close()` joins the worker on the caller's thread: 98 ms measured during a 4K refill, up to 3 s by design.  Core calls it on the render thread (`video_deck_layer.py:403`) |
| P3-11 | `deck_video_source.py:100`, docstring `:65-78` | The drop-in's default long edge is 960 and its memory math is written for 960; core passes 720 (`video_deck_layer.py:111`).  At 720 the ring is 121 frames, 106 MB per deck — VERIFIED |
| P3-12 | `deck_video_source.py:50-53` | `frame_for()` is described as allocation-free; it builds a list and a lambda per call.  Measured cost is 26 µs, so the claim is wrong but harmless |
| P3-13 | `docs/drop-ins.md:67` | The registry still says the Video Player uses ffpyplayer |
| P3-14 | `unicornviz/video_deck_layer.py:430` | Core's deck texture is 3-channel RGB, the slow upload format on this GPU (see below).  Core already avoids the `tobytes()` copy that both effects make |

## Optimization options, measured

Ranked by payoff.  Each figure is from this machine on the test clips.

**Deck video source**

1. **Seek to the target, not 2 s before it, and convert only frames that
   will be kept.**  Cue jumps drop from 0.4–2.5 s to 35–660 ms (table in
   P1-2).  Counting the cap only on kept frames also removes P1-1.
2. **Fast-bilinear scaling.**  10.3 → 3.8 ms per 1080p frame; roughly a
   third of the worker CPU and refill time.
3. **Fill behind the playhead when the rate is negative**, before the ring
   edge is reached.  Not measured.

**Video Clips**

1. **Decode on a worker thread** and leave only the upload on the render
   thread.  The pattern already exists in the Video Player's `_Decoder`.
   Render-thread cost per clip frame falls from 7.2 to 3.6 ms at 720p and
   from 11.1 to 7.0 ms at 1080p before any format change.
2. **Open and position the next clip on the worker before it is needed.**
   Removes the 150–200 ms clip-change hitch and the activation seeks.
3. **Close clips that are no longer in either slot.**  Caps memory at two
   captures (about 130 MB) regardless of group size.
4. **Drop the flip, the color conversion and the copy** (flip in the shader,
   swizzle or convert once, pass the array directly).  Saves 1.3 ms per clip
   frame at 720p, 2.5 ms at 1080p, 8.1 ms at 4K.

**Video Player**

1. **Drain the audio queue when there is no stream.**  Fixes P2-5.
2. **Advance the clock between audio callbacks** from wall time.  Fixes P2-6.
3. **Convert in the decoder to what the screen needs** (output-sized, or
   YUV planes).

**Texture upload format (all three, and core's deck layer)**

Median upload time for one frame:

| Format | 720p | 1080p | 2160p |
|---|---|---|---|
| RGB, 3 channels (current everywhere) | 4.77 ms | 6.61 ms | 20.49 ms |
| RGBA, 4 channels | 1.99 ms | 3.80 ms | 6.79 ms |
| RGBA through a pixel buffer object | 4.70 ms | 8.58 ms | 12.20 ms |
| YUV 4:2:0 as three single-channel planes | 1.52 ms | 1.18 ms | 4.65 ms |

Uploading the decoder's native YUV planes and converting in the fragment
shader is 3 to 5 times faster than the current upload, and also removes the
decoder-side RGB conversion (1.3 ms at 720p, 1.8 ms at 1080p, 3.3 ms at 4K
per frame in PyAV).  Plain RGBA is a smaller change for a 2 to 3 times gain.
A pixel buffer object did not help on this driver.  These ratios are for
Mesa on Intel; a discrete GPU may differ.

## What works

- **Forward deck playback.**  100% fresh frames at 720p and 1080p at 1× and
  1.5×, 8 ms median frame error, no seeks.
- **Scratching inside the cache.**  100% fresh, no seeks.
- **Deck lookups.**  `frame_for()` costs 26 µs; a paused deck or one sitting
  at end of file uses 1–3% of a core.
- **Non-blocking open.**  The deck source and the Video Player both open
  files off the render thread; the Video Player's `_init` takes 2 ms.
- **Video Player A/V sync at 720p and 1080p** with a small audio block:
  under 25 ms, no dropped frames.
- **Teardown.**  Both effects release their captures, threads, handles and
  textures on `destroy()`.
- **Video Clips metadata cache.**  A second activation costs 3–4 ms.

## What this audit did not do

- Launch the app, open a window, or open a real audio stream.  The Video
  Player's audio behavior is from a simulated device.
- Decode the owner's clips, or test any deck video from the music library.
- Test hardware decoding, Windows, or a discrete GPU.
- Test the mixer side of music-video decks (position publishing, latency
  reporting); only the frame cache and core's upload call were in scope.
- Change any code.  The "optimization options" are measurements of
  alternatives, not patches.
