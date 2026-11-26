# Effects packs and projectM audit (2026-11-26)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-26

Scope: the 13 `effects-*` drop-ins (68 effects, 80 files, about 28,000 lines)
and `projectm-01` (`0.12.8`, about 2,600 lines plus two shell scripts), at the
submodule commits pinned by master `a81d82b`.

| Pack | Version | Effects | Last commit |
|---|---|---|---|
| effects-games | 0.18.0 | 12 | 2026-09-13 |
| effects-rollercoast | 0.5.1 | 6 | 2026-09-24 |
| effects-flying | 0.4.0 | 12 | 2026-09-09 |
| effects-ukiyo-e | 0.4.0 | 4 | 2026-09-11 |
| effects-tech | 0.11.0 | 6 | 2026-09-16 |
| effects-retro | 0.9.1 | 6 | 2026-09-05 |
| effects-particles | 0.10.0 | 3 | 2026-09-16 |
| effects-vector | 0.10.0 | 4 | 2026-09-05 |
| effects-cosmic | 0.11.2 (README) / 0.11.1 (code) | 5 | 2026-09-03 |
| effects-feature | 0.9.0 | 3 | 2026-07-13 |
| effects-immersive | 0.10.1 | 3 | 2026-08-08 |
| effects-psychedelic | 0.9.0 | 3 | 2026-09-05 |
| effects-holiday | 0.5.1 | 1 | 2026-08-06 |

**Every pack was last committed before the 2026-09-30 effects audit, so every
finding of
[2026-09-30-effects-and-cameras-bug-audit.md](2026-09-30-effects-and-cameras-bug-audit.md)
located in these packs is still open**, except E1, whose cause was changed in
core (see P2-1).

Method:

- Every Python file and every embedded shader was read (`corkscrew.py` and
  `night_coaster.py` as complete diffs against `first_drop.py`).
- **Harness:** all 68 effects were constructed and run in an offscreen GL
  context: 240 frames with synthetic music, silent audio, non-finite audio,
  frame times of 0 s, 0.1 s and 1/144 s, a resize, a destroy with a GL-object
  leak count, and a two-instance comparison at equal time.
- **Games:** each of the 12 simulations ran 30 simulated minutes × 6 runs at
  60 fps (silent, default, busy) plus 10 minutes × 18 runs at mixed frame
  times, with stall detection.  Breakout and Joust got seeded hunts of 120
  and 96 runs.
- **Travel effects:** wraps were stepped across and measured as the frame
  difference at the wrap against a normal frame step; speed changes were made
  through core's `apply_speed`.
- **projectM:** libprojectM is installed on this machine, but the real effect
  was **not** constructed (it would load the library without an SDL context
  and write state into the checkout).  It was audited by reading, with its
  pure-Python catalog, state and file operations exercised on a scratch
  preset tree.  No preset switch was timed.  Neither shell script was run.
- **GPU:** every timing is on **Mesa Intel Iris Xe** (the only GPU on this
  machine), with other audit jobs sharing it.  The project's "8 ms on a GTX
  1060" rule cannot be judged from these numbers.
- Nothing was rendered at 21:9 or 4:3 except where stated; aspect findings
  are by reading.
- **VERIFIED** = reproduced by script.  **SUSPECTED** marks what could not be
  confirmed.

Scripts and images: `/var/tmp/uv-fx-audit/{mine,games,travel,small,pm}/`.

Severity: **P1** an effect that can crash the app, raise every frame, go
blank for the rest of its time on air, or delete files outside its own tree.
**P2** visibly wrong behavior an audience would notice, or files deleted
without confirmation.  **P3** latent, cosmetic or a rule violation.

---

## Baseline

| Check | Result |
|---|---|
| Harness, 68 effects, normal running | No exception, no GL error, no blank or non-finite frame, no leaked GL object, at any tested frame time or on resize |
| Embedded shaders | 140 found, 140 compile (`glslangValidator`) |
| bandit (medium and above) | nothing |
| `update()` overrides | all call `super().update()` |
| projectM tests | 39 passed with one deselected (see P3-48) |
| Tests for the effects packs | none: no pack has a `tests/` directory |

---

## Summary

| ID | Sev | Where | One line |
|---|---|---|---|
| P1-1 | P1 | projectM | Delete in the preset manager follows symlinks and deletes the target outside the preset tree — **VERIFIED** (needs a symlinked preset) |
| P2-1 | P2 | core + 40 effects | Speed changes jump the picture: core's `apply_speed` fix has no matching constants in any pack — **VERIFIED** |
| P2-2 | P2 | rollercoast | The rides' `march_steps` quality setting makes all six fail to construct at any non-default value — **VERIFIED** |
| P2-3 | P2 | rollercoast | Five of six rides exceed the frame budget alone at 1080p on this GPU — **VERIFIED** |
| P2-4 | P2 | rollercoast | Corkscrew's first inversion is a snap, and on a fifth of activations not a roll — **VERIFIED** |
| P2-5 | P2 | flying | Canyon Run puts the camera inside the rock for 3-11 % of frames — **VERIFIED** |
| P2-6 | P2 | flying | Nebula Drift replaces the whole nebula about every two minutes and loses its stars — **VERIFIED** |
| P2-7 | P2 | flying | Space Dance layers pop out at full brightness every 3-4 s — **VERIFIED** |
| P2-8 | P2 | flying | Portal Flight flashes the full frame 2-9 times a second — **VERIFIED** |
| P2-9 | P2 | flying, rides, ukiyo-e, games, cosmic | Wrap pops: 20 measured wraps jump the picture 3× to 154× a normal frame step — **VERIFIED** |
| P2-10 | P2 | vector | 3D Cube and Vector multiply their matrices in the wrong order: no cube, no polyhedra — **VERIFIED** |
| P2-11 | P2 | games | Super Mario's backdrop never scrolls: it slides one tile and snaps back — **VERIFIED** |
| P2-12 | P2 | games | Mario Kart's sun and hills teleport once per lap — **VERIFIED** |
| P2-13 | P2 | games | Breakout locks into a closed orbit when audio is constant — **VERIFIED** |
| P2-14 | P2 | games, vector | Joust's wings and Laser Tunnel's rings are scrambled by time × audio — **VERIFIED** |
| P2-15 | P2 | retro | Fractal Zoom is pixelated for the last quarter to third of every zoom cycle — **VERIFIED** |
| P2-16 | P2 | particles | Particle Storm collapses to about 24 distinct particles within two minutes — **VERIFIED** |
| P2-17 | P2 | tech | Hacker Terminal 2.0 never shows text — **VERIFIED** |
| P2-18 | P2 | tech | Tron Grid flies the camera through its own cubes about 20 % of the time — **VERIFIED** |
| P2-19 | P2 | particles | Fireworks forces a window-size viewport on the caller's target — **VERIFIED** |
| P2-20 | P2 | projectM | Permanent deletes and the bulk move have no confirmation; undo does not move files back — **VERIFIED** |
| P2-21 | P2 | projectM | Bulk enable/disable writes snapshot files into the submodule — **VERIFIED** |
| P2-22 | P2 | projectM | projectM renders with no GL state saved or restored and no viewport passed |
| P2-23 | P2 | projectM | Preset loads compile on the render thread, including one at every activation |
| P2-24 | P2 | projectM | A non-UTF-8 preset file name raises at the native boundary — **VERIFIED** |
| P2-25 | P2 | projectM | The effects-browser thumbnail shares the on-air effect's native instance (SUSPECTED) |
| P3-1…P3-50 | P3 | all | See the P3 tables |

---

## P1

### P1-1. projectM's delete follows symlinks out of the preset tree — VERIFIED

`drop-ins/projectm-01/projectm_effect.py:1528-1543`.  `delete_presets()`
resolves each catalog path before acting on it:

- **Shift+Delete** in the manager (`unicornviz/hotkeys.py:1583`) calls
  `source.unlink()` on the resolved path.  For a `.milk` file that is a
  symlink into the preset tree, the real file elsewhere is deleted
  permanently and the dangling link stays.
- **Delete** moves the real file into `preset-trash/`.

Reproduced on a scratch tree: "outside target still exists: False" for both.
The single-preset `delete_preset()` (1482) unlinks only the link.

The precondition is a preset that is a file symlink.  Directory symlinks are
not scanned on Python 3.14.  Whether the owner's preset library contains file
symlinks was not checked (the drop-in's own `textures/` directory holds 20
committed symlinks with absolute targets, so the pattern is in use nearby).

There is no confirmation on either key (P2-20).

**Fix:** act on `item.path`, and refuse when `resolve()` leaves the preset
roots.

---

## P2

### P2-1. Speed changes jump the picture — VERIFIED

`unicornviz/effects/base.py:282-314` (core) and every pack.

The 09-30 audit's E1 was "speed hotkeys hard-cut 34 effects".  Core now
routes every speed write through `apply_speed`, which rescales `effect.time`
so that `time × (SPEED_TIME_BIAS + SPEED_TIME_SCALE × speed)` stays
continuous.  The defaults are bias 0, scale 1: correct for a shader that
computes `iTime * iSpeed`.

No effect in these 13 packs or in projectM declares either constant.  Two
groups are therefore wrong:

**(a) Shaders that use `iTime * (a + b * iSpeed)` still cut** (10 effects):

| Effect | Constants needed | Frame step across a speed change vs normal |
|---|---|---|
| Space Dance | 0.3, 0.4 | 50× |
| Cloud Bounce | 0.25, 0.35 | 9.3× |
| Space Tunnel | 0.5, 0.8 | 5.8× |
| Bass Hammer | 0.6, 0.9 | 5.5× |
| Vector Hallway | 0.6, 0.7 | 3.0× |
| Sun Ship 3000 | 0.6, 0.5 | 12.7× |
| Reactor Breach | 0.8, 0.5 | 7.3× |
| Threat Matrix | 0.8, 0.5 | 2.9× |
| Laser Tunnel | 0.9, 0.6 | shader time moved 900 s |
| Cathedral of Bass | none fits (two uses of time) | 3.1× |

With the constants set on the instance, the five flying effects measured
1.1-1.3×.

**(b) Effects that integrate speed on the CPU and never multiply time by it
now jump, where before they did not** (about 30 effects): all six rides, the
eleven 2.0 flying and ukiyo-e effects, eleven of the twelve games, ANSI
Viewer, Hacker Terminal 2.0, Particle Storm, Fireworks, and projectM (whose
clock is `self.time`, so a speed change from 1 to 2 halves it).  Every
speed write moves `effect.time` (6,100 → 4,693 for a 1.3× change), and
anything driven by time jumps:

| Effect | Step across `apply_speed` × 1.3 vs normal |
|---|---|
| Nebula Drift | 24× |
| Joust | 31× |
| Log Flume | 6.7× |
| Raijin's Drums | 4.6× |
| Warp Drive | 3.7×-4× |
| First Drop | 2.5× |

Auto VJ retargets speed continuously, so this is not limited to the hotkeys.

Correct today: the effects that use plain `iTime * iSpeed` (Tunnel,
Starfield, Plasma, Kaleidoscope, Psychedelic, Hexy Stars and others), and
Cosmos, Wavey Gravy and Rainbow Trance.

**Fix:** declare the two class constants per effect: the table values for
group (a), `1.0` / `0.0` for group (b).  A safer core default would be to
rescale only when an effect opts in.

### P2-2. `march_steps` breaks every ride — VERIFIED

`effects-rollercoast/first_drop.py:124-126`, `524-527`, `546-550` and the
same block in the other five rides.  The shader's uniform arrays are
hard-coded (`iP[47]`, `[31]` in Mine Train, `[41]` in Log Flume) while only
`#define STEPS` follows the parameter.  `_build_march()` then writes a
`steps + 1` table into the fixed-size uniform and moderngl raises
`invalid uniform size` inside `_init()`, after the program and buffers exist
(they leak).

24 of 24 combinations fail (6 rides × 12 / 24 / 64 / 96).  The value reaches
`_init` from the Effects tab or `[effects.<Ride>] march_steps`, documented as
12-96.  It is the rides' only quality control.

**Fix:** size the arrays from the define (`iP[STEPS + 1]`).

### P2-3. The rides exceed the frame budget on this GPU — VERIFIED

Render time, mean / p99 ms, Intel Iris Xe:

| Ride | 1080p | 4K |
|---|---|---|
| Night Coaster | 27.8 / 29.8 | 84.7 / 89.5 |
| Log Flume | 23.4 / 25.5 | 91.8 / 96.9 |
| First Drop | 21.3 / 23.7 | 46.4 / 48.3 |
| Mine Train | 19.6 / 20.1 | 76.7 / 79.9 |
| Corkscrew | 15.6 / 16.6 | 46.9 / 50.8 |
| Coaster Cam | 13.3 / 13.9 | 42.0 / 42.7 |

Each figure includes 1.6-2.5 ms of allocating numpy work in `render()`
(`_build_march`).  A second effect renders during every transition, so a
scene change into or out of a ride drops frames here.  There is no
resolution divisor, and the one quality knob is P2-2.  What a discrete GPU
does was not measured.

Other effects above 8 ms at 1080p on this GPU: Reactor Breach (13.3),
Starfield while a beat warp is active (9.5-10.0, against 1.7 otherwise).

**Fix:** repair P2-2; add a resolution-scale parameter; move `_build_march`
into `update()` with preallocated arrays.

### P2-4. Corkscrew's first inversion — VERIFIED

`effects-rollercoast/corkscrew.py:438-467`, `579-582`.  The roll is applied
about the world Z axis, not the track tangent.  Over 300 seeds: in 21 % the
first inversion nets under half a revolution; the median peak roll rate is
0.74 rad/m against a designed 0.08; the 90th percentile is more than 90° in
one frame.  The second inversion is clean.

The same `track_frame` in all six rides banks toward world +x, so the track
leans out of the turn on 42-54 % of a lap (P3-1).

**Fix:** rotate a horizon-level lateral about the tangent by `roll`.

### P2-5. Canyon Run flies inside the rock — VERIFIED

`effects-flying/canyon_run.py:366-371`.  The walls use `centre(d) * 1.6`; the
ship offset subtracts `centre(dist)` without the 1.6.  When the canyon
snakes wide the ship is outside the walls and every ray hits at once: a flat
grey-green screen for 3.0 % of frames with moderate audio and 10.6 % with
loud mids, in spells of up to 2.6 s.

**Fix:** subtract `self._centre(dist) * 1.6`.

### P2-6. Nebula Drift — VERIFIED

- `effects-flying/nebula_drift.py:259-260`: `_drift % 100` shifts the noise
  domain by (35, −22) at the wrap, replacing the picture in one frame
  (33.9 against 0.22 normal, 154×), roughly every two minutes.
- `:263-266`: star x wraps, star y never does.  Stars with |y| < 0.5 average
  4.7 of 7 after one minute, 1.8 after three, 0.5 after ten.  The stars are
  the light source and the beat event.

**Fix:** wrap each derived offset at the noise period; wrap y as x.

### P2-7. Space Dance layer pops — VERIFIED

`effects-flying/space_dance.py:95-119`.  Each gas and star layer wraps its
depth where its fade is still 0.61 and its gain is 4, so it vanishes at full
brightness: 54× to 336× a normal frame step, one pop every 3-4 s.

**Fix:** make the near fade reach zero at the wrap.

### P2-8. Portal Flight whites out — VERIFIED

`effects-flying/portal_flight.py:185`, `266-275`.  Every gate passage sets a
full-frame white flash.  Flashes per second: 2.25 at speed 1, 4.5 at speed 2,
8.9 at speed 4; mean pixel 146, 186 and 218 of 255.  Auto VJ raises speed
toward 4.0 in drops.  Above about 3 flashes a second this is a
photosensitivity concern as well as a wash-out.

**Fix:** a minimum interval between flashes; confine the flash to the gate.

### P2-9. Wrap pops — VERIFIED

CPU phases wrapped at a round number, and shader `mod(iTime, N)`, feed
functions that are not periodic in that number.  The 09-30 audit's E4 is
unchanged; this pass measured each wrap as the frame difference at the wrap
against a normal step.

| Effect | Wrap | Ratio |
|---|---|---|
| Nebula Drift | `_drift % 100` | 154× |
| Space Dance | `_depth % 1000` | 81× |
| Mine Train | `_flow % 100` | 72× |
| Floating World | `_wind % 100` | 43× |
| Q*bert | time 600 s | 27× (80 % of pixels) |
| Frogger | time 600 s | 21× |
| Onden Watermill | `_current % 100` | 18× |
| Portal Flight | `_travel % 100` | 14× |
| Cosmos | `mod(t, 1024)` | 13.9× |
| Sun Ship 3000 | `_mid_phase % 1000` | 13.5× |
| Warp Drive | `_warp % 100` | 12.8× |
| Asteroid Run | `_travel % 100` | 10.8× |
| Raijin's Drums | time 600 s | 8.6× |
| Bass Hammer | `mod(iFlight, 248)` | 7.9× |
| Night Parade | `_march % 64` | 6.5× |
| Wingsuit Dive | `_travel % 100` | 5.1× |
| Canyon Run | `_travel % 100` | 4.3× |
| Cloud Surfer | `_run % 400` | 4.1× |
| All six rides | time 300 s | 3.2×-4.4× |
| Van Gogh | `mod(t, 1024)` | 2.9× |

Night Parade and Mine Train were recorded as clean on 09-30; they are not.
Vector Hallway and Space Tunnel are clean.

**Fix:** wrap each derived term at a period of the function that consumes
it, or integrate the phase on the CPU and pass it already reduced.

### P2-10. 3D Cube and Vector — VERIFIED

`effects-vector/cube_3d.py:186-203`; `vector.py:171-178`, `280-287`.  The
projection matrix is laid out transposed, multiplied with conventional model
matrices, and uploaded with `.write(tobytes())`, so the transforms apply in
the wrong order.

- **3D Cube:** a mean of 4.0 of 8 corners behind the camera in 40 of 40
  poses; frame coverage swings from 2 % to 100 %.  The picture is slabs and
  slivers.  The conventional product gives a clean cube at a steady 32-37 %
  (`games/cube_shipped.png` against `cube_conventional.png`).
- **Vector:** 33 % of wire vertices are behind the camera; the three
  polyhedra read as long lines across the frame.

**Fix:** build `P.T @ T @ S @ R` and upload its transpose.

### P2-11. Super Mario's backdrop — VERIFIED

`effects-games/super_mario.py:205-302`.  Every backdrop layer is computed
from `iCamFrac + uv.x * 15`; the absolute camera tile never reaches the
shader.  Hills, clouds and shrubs slide one tile at foreground speed and snap
back, five to six times a second.  With a flat world and no entities, frames
1, 2 and 7 tiles apart are bit-identical.

**Fix:** pass the camera tile, wrapped per layer, at each layer's parallax
rate.

### P2-12. Mario Kart's sky — VERIFIED

`effects-games/mario_kart.py:171`, `187`, `529-532`.  `sin(heading * 0.5)`
and `heading * 1.6` are not periodic in 2π, and the heading wraps at ±π.  A
0.004 rad turn across the wrap moves the sun from x = 0.685 to 0.298, every
30.7 s in silence and every 15.7 s with busy audio.

**Fix:** accumulate an unwrapped heading.

### P2-13. Breakout's closed orbit — VERIFIED

`effects-games/breakout.py:284-344`.  With steady audio the paddle and
reflection are fully deterministic, and the ball can settle into a repeating
path that never reaches the last bricks.  In 120 seeded 30-minute runs, 6
ended in an unbroken drought of 955-1,492 s; one seed extended to two hours
was stuck from 247 s to the end, in an exact 8-bounce cycle.

It needs constant audio (silence, a dropout, a changeover): none of 40
busy-audio runs stalled, and the stuck run recovered 1.7 s after the audio
changed.  The ball keeps moving; the board never clears.

**Fix:** a small `self.rng` jitter on the reflection, or a no-hit watchdog.

### P2-14. Time × audio (E3, still open) — VERIFIED

- **Joust** `effects-games/joust.py:464`, `251`: the wing phase is
  `sin(time * (5 + flap * 8 + mid * 3))`.  At time 5,000 the wing flips sign
  12.1 times a second in silence and 29.9 with music.
- **Laser Tunnel** `effects-vector/laser_tunnel.py:132`, `142`: a 0.01 bass
  change gives a frame step of 12.6 against 2.2.
- **Rainbow Trance** `effects-feature/rainbow_trance.py:119`, `152-153`:
  5.4×.
- Also by reading: Galaga's warp starfield (`galaga.py:135-137`) and Van
  Gogh's stars (`van_gogh.py:122`, about 1,000 cells per frame at each flow
  retarget).

No such line exists in the rollercoast, flying or ukiyo-e packs.

**Fix:** integrate the phase in `update()`.

### P2-15. Fractal Zoom runs out of float32 — VERIFIED

`effects-retro/fractal_zoom.py:78`, `187`, `237`.  The zoom ceiling is 8e5 to
3.2e6; float32 stops resolving adjacent pixels well before.

| Zoom | Mean block width at 1080p |
|---|---|
| 1e4 | 1.1 px |
| 1e5 | 2.4 px |
| 3e5 | 5.5 px |
| 8e5 | 13 px |
| 3.2e6 | 54 px |

That is 27-39 s of each 116-128 s cycle, and sooner at 4K.

**Fix:** cap the ceiling near 3e4-5e4, or pass the center as high/low pairs.

### P2-16. Particle Storm collapses — VERIFIED

`effects-particles/particle_storm.py:89`, `49-51`.  Each respawn sets
`out_seed = rng(in_seed + 99.9)`, which in float32 is heavily many-to-one.
All 150,000 particles converge onto a few seeds.

| Time on air | Distinct seeds | Lit screen fraction | Mean brightness |
|---|---|---|---|
| 1 s | 99,663 | 0.69 | 92 |
| 10 s | 372 | 0.25 | 40 |
| 60 s | 51 | 0.06 | 12 |
| 120 s | 24 | 0.03 | 6 |

It stays there for the rest of a 30-minute run.  No NaN and no pool drain.

**Fix:** keep the seed's entropy (mix in time, or carry an integer hash).

### P2-17. Hacker Terminal 2.0 shows no text — VERIFIED

`effects-tech/hacker_terminal_v2.py:290-313`.  It requires a 32,768-byte
glyph atlas; `assets/fonts/font8x16.bin` is 4,096 bytes (1 bit per pixel), so
it always takes the solid-white fallback.  Every activation is a full-screen
flickering phosphor wall with no glyphs.  The candidate paths are also
relative to the working directory, and one names a directory that does not
exist.

**Fix:** build the atlas through the core CP437 font loader.

### P2-18. Tron Grid flies through its cubes — VERIFIED

`effects-tech/tron_grid.py:82-92`, `135-139`.  Cubes sit at x = 0 every 8
units; the camera path is x = 1.6·sin(0.22 t).  Whenever |x| < 0.5 (about
20 % of the time) a cube face fills the screen: the image freezes for about
nine frames, then hard-cuts on exit.

**Fix:** offset the cube lattice from the camera lane.

### P2-19. Fireworks overrides the viewport — VERIFIED

`effects-particles/fireworks.py:529-574`.  It sets
`ctx.viewport = (0, 0, self.width, self.height)` regardless of the bound
target.  With `render.internal_scale < 1` only the bottom-left of the picture
is shown; inside Candy Frame it ignores the frame rectangle (09-30 E16).
Its `candy_frame_disallow = True` has no effect: core's
`_effect_requests_frame_scaling` returns True for every effect.

**Fix:** save and restore the viewport; size the accumulation FBOs from it.

### P2-20. projectM: no confirmation, and undo does not undo — VERIFIED

`projectm_effect.py:1482`, `1534`, `2233`; `unicornviz/hotkeys.py:1532-1588`.
One Shift+Delete unlinks a preset.  One Shift+Q on "! Disabled" moves every
disabled preset out of `presets/`; after an "isolate category" that is
nearly the whole library.  There is no bulk restore, and Ctrl+Z restores the
state without moving the files back ("files back in presets/: 0").  The
drop-in's own `docs/planning.md` asks for confirmation before any filesystem
change.

**Fix:** a confirm step for Shift+Delete and Shift+Q; make undo move the
files or refuse.

### P2-21. projectM dirties its own submodule — VERIFIED

`projectm_effect.py:888-890`.  Bulk enable / disable / isolate writes
`states/state-*.json` inside the drop-in.  `.gitignore` ignores
`preset_manager_states/`, not `states/`.  Any use of those keys leaves
untracked files in the submodule, in every seat and in the live checkout.

**Fix:** write under `APP_ROOT/runtime/projectm/`.

### P2-22. projectM and GL state

`projectm_effect.py:1780-1787`, `344-349`.  The effect hands the current
framebuffer to `projectm_opengl_render_frame_fbo` and returns.  Nothing is
saved or restored: viewport, scissor, program, VAO, texture bindings, blend,
depth.  projectM sizes itself from the window size.

By reading: with `render.internal_scale < 1` the target is smaller than
projectM's size; inside Candy Frame only the scissor clips it; and moderngl's
cached state may not match GL when the next effect of a transition draws.
The drop-in's `PHASE1-PLAN.md` lists this as an unresolved risk.  The
on-screen result was not run.

**Fix:** render into a private FBO sized from the viewport and blit; re-bind
the caller's FBO and reset program, VAO and blend.

### P2-23. projectM preset loads on the render thread

`projectm_effect.py:1588-1611`, `625`.  `projectm_load_preset_file` is
synchronous (the file's own comment says so).  `_init()` loads a start preset
at every scene change into projectM, and every Up/Down in the manager loads
immediately.  Not timed.

**Fix:** defer the start preset until the transition ends; debounce the
manager preview.

### P2-24. projectM and non-UTF-8 file names — VERIFIED

`projectm_effect.py:329-334`.  A preset whose file name is not valid UTF-8
makes `next_preset()` raise `UnicodeEncodeError` out of `update()`; core
quarantines the whole effect class after the second occurrence.

**Fix:** `os.fsencode(path)`; catch per-preset errors around the load.

### P2-25. projectM's shared native instance (SUSPECTED)

`unicornviz/app.py:10898`; `projectm_effect.py:606-625`.  A thumbnail-size
`ProjectMEffect` shares the one native instance, so its `_init` would resize
the window, install its own callbacks and load a random preset into the
bridge the on-air effect is using.  By reading; whether the effects browser
excludes projectM from thumbnails was not checked.

---

## P3

### Rides, flying, ukiyo-e

| ID | Where | What |
|---|---|---|
| P3-1 | `first_drop.py:462-468` (all six rides) | Banking leans toward world +x: out of the turn on 42-54 % of a lap, up to 36° the wrong way — **VERIFIED** |
| P3-2 | `first_drop.py:248-249`, `563-567`; `log_flume.py:271-323` | Support bents and ties shift at the lap seam (the spacing does not divide the circuit); Night Coaster's fireworks relocate |
| P3-3 | `corkscrew.py:457` | `roll_force` leaves the ride tilted unless it is a multiple of 0.5: 197° at 1.3 — **VERIFIED** |
| P3-4 | `night_parade.py:458`, `468` | Raises `AttributeError` if rendered before its first update — **VERIFIED** (latent: core updates first) |
| P3-5 | `night_parade.py:389`, `445` | The parade walks backwards on half its activations: silhouettes always face +x |
| P3-6 | `onden_watermill.py:462`, `514-523` | 13 splashes per revolution from 12 buckets — **VERIFIED** |
| P3-7 | `floating_world.py:377` | The claw foam flips sides whenever the crest crosses the center — **VERIFIED** (4×) |
| P3-8 | `canyon_run.py:188-197`, `211`, `352-355`; `wingsuit_dive.py:132-194` | Rock and ground textures are glued to the camera (the shader never receives distance flown); sparks and the rim line are misplaced |
| P3-9 | `mine_train.py:202-204` | A noise seam left of the bore center — **VERIFIED** in isolation |
| P3-10 | `floating_world.py:505-506`; `onden_watermill.py:402-403` | Zoomed out, the picture shrinks to a stamp: 90 % of pixels near-black at zoom 0.3 — **VERIFIED**.  Neither publishes `random_zoom_min` |
| P3-11 | `raijin_drums.py:365`; `night_parade.py:387`; rides; `asteroid_run.py:294`; `floating_world.py:800` | Dead or misleading parameters: `glitter` and `neon_gain` cached at init; `march_steps` read only at init; `field_density` works backwards |
| P3-12 | rides, Nebula Drift, Cloud Surfer, ukiyo-e | Shader compile at activation: 200-450 ms with a cold driver cache — **VERIFIED**; 4-40 ms warm |
| P3-13 | `onden_watermill.py:171`, `240`; `nebula_drift.py:265`; `night_parade.py:63` | Fixed compositions at other aspects: Fuji off-screen at 4:3; wrap edges visible at 21:9 and 32:9 |

### Games, vector, retro

| ID | Where | What |
|---|---|---|
| P3-14 | `donkey_kong.py:696` vs `534` | Barrels spawn nearer the damsel than Kong on 97 of 200 activations — **VERIFIED** |
| P3-15 | `vector.py:125-138` | The wire sparkle is dead about 95 % of the time (hash input bound too loose) — **VERIFIED** |
| P3-16 | `joust.py:434-507` | At a sustained 10 fps Joust never finishes a wave (16 of 16 runs) — **VERIFIED**; fine at 30 fps and above |
| P3-17 | `joust.py:480-488` | The player is in the invulnerable blink for 43-48 % of frames — **VERIFIED** |
| P3-18 | `galaga.py:600-613`, `428-431` | In a portrait window Galaga cycles 134 waves a minute — **VERIFIED**; the "boss rank" pattern has no bosses |
| P3-19 | `super_mario.py:821-858` | The hero and walkers run through pipes |
| P3-20 | `frogger.py:493`, `631-633`, `721` | Under music the frog gets home 5-14 times against 101-106 deaths per 30 minutes — **VERIFIED**; 20 % of activations still simulate more logs than the shader draws (09-30 E6, E14) |
| P3-21 | `neon_pac.py:835` | Ghosts occupy wall cells for hundreds of frames per run (09-30 E12, re-verified) |
| P3-22 | `ansi_viewer.py:143-202` | A failed load leaks a texture and shows black; an empty directory is black; loading costs 121-712 ms per file on the render thread; uses the global `random` (09-30 E20, re-verified) |
| P3-23 | `tetris.py:479-503`; `neon_pac.py`; `donkey_kong.py` | Per-frame cost spikes: Tetris plans a move in 3.5 ms about 3.7 times a second; Neon Pac loads a board in 2.4 ms inside `update()` |
| P3-24 | `frogger.py`; `super_mario.py` | Aspect: Frogger's home bays are clipped at 4:3; Super Mario's question blocks are off-screen at 21:9 |
| P3-25 | `breakout.py:127-129`; `disco_ball.py:406-408`; `dali.py:90`; `missile_command.py:105` | Other 09-30 items still open in these packs: oval ball (E11), Disco Ball wraps (E4), hash collapse (E2), time × audio (E3) |

### Small packs

| ID | Where | What |
|---|---|---|
| P3-26 | `tunnel.py:94`, `68` | A hard color seam along the −x axis — **VERIFIED** |
| P3-27 | `fireworks.py:32`, `431-443` | Under busy music 97.6 % of new sparks overwrite a visible one: bursts vanish instead of fading — **VERIFIED** |
| P3-28 | `wormhole.py:86-108` | An opaque shell hides the crystal core and halo; `shard_count` is dead and the march pays for hidden geometry — **VERIFIED** |
| P3-29 | `starfield.py:146-151` | A 180-iteration per-pixel loop runs for half a second after every beat: 1.7 → 9.5 ms — **VERIFIED** |
| P3-30 | `cosmos.py:86-93` | Stars are 0.36 px in radius at 1080p; at one sampled instant none was lit — **VERIFIED** |
| P3-31 | `plasma.py:81-94` | `render()` before the first `update()` raises (latent) — **VERIFIED** |
| P3-32 | `particle_storm.py:298-337` | The picture is one simulation frame behind; the beat relocation of emitters is overwritten the next frame (09-30 E15, re-verified) |
| P3-33 | `alien_invasion.py:198`; `cyber_war.py:141-165`; `hacker_terminal.py:114-148`; `starfield.py`; `sun_ship_3000.py:186` | Dead or mis-wired parameters: `radar_sweep`, three Cyber War knobs, `text_speed`, `noise_intensity`, `seed` (E19); Cyber War's "nodes" are 9 single texels; Starfield's parallax layers do not move |
| P3-34 | `america_250.py:190` | `PING_PONG_FRIENDS` names "Grand Finale", which is a controller, not an effect |
| P3-35 | `hacker_terminal.py:8`, `176` | Uses the global `random` module |

### Cross-cutting

| ID | What |
|---|---|
| P3-36 | **Randomization:** eight effects render identical frames when two instances are set to the same time, so nothing but the random start time varies between activations — Cosmos, Escher, Hexy Stars, Psychedelic, Rainbow Trance, Tron Grid, Wavey Gravy, Copper Bars — **VERIFIED**.  Cyber War differs by 0.00004.  Copper Bars always opens in the same mode for its first 15 s |
| P3-37 | **Non-finite audio:** a fully NaN FFT (what the core analyzer can emit, core audit P2-4) breaks no effect.  Other non-finite fields do: an infinite `bass_n` makes Cube3D and Vector raise on every frame from then on and leaves Fractal Zoom blank; NaN in `audio.bands` leaves Frogger blank; nine small-pack effects stay black or non-finite after one bad frame (Rainbow Trance, Tunnel, Kaleidoscope, Hacker Terminal 2.0, Cathedral of Bass, Reactor Breach, Threat Matrix, Sun Ship 3000, Fireworks) — **VERIFIED**.  That the app can produce these inputs was not shown |
| P3-38 | **Version sync:** `effects-cosmic` README says 0.11.2, its only `__version__` says 0.11.1.  `__version__` is scattered across several files per pack with no primary module |
| P3-39 | **Shader length:** 31 fragment shaders exceed the 150-line guideline (all 12 games, 13 of the 22 travel effects, Disco Ball, Wormhole, Starfield, Tron Grid, Rainbow Trance, America 250).  The longest: Super Mario 534, Floating World 433, Mine Train 390 |
| P3-40 | **`render()` allocations:** the rides' `_build_march`, seven games, 3D Cube, Vector, Metaballs, Hacker Terminal 2.0, several flying 2.0 effects |
| P3-41 | **Stale READMEs:** `effects-flying` ("flying-01", "all seven are pure GLSL"; there are twelve), `effects-ukiyo-e` ("both"; there are four), the games, vector and retro READMEs titled `*-01`, `effects-immersive` describing a different Wormhole.  The six ride files are about 560 identical lines each |

### projectM

| ID | Where | What |
|---|---|---|
| P3-42 | `projectm_effect.py:540`, `1742` | Every speed change teleports projectM's clock (part of P2-1) |
| P3-43 | `projectm_effect.py:813-876`, `1035-1067` | State file: a corrupt or wrong-shape file is treated as empty and overwritten — **VERIFIED**; entries are absolute paths; a hand-edited undo snapshot raises on Ctrl+Z.  The running app writes under `runtime/`, not the committed file |
| P3-44 | `projectm_effect.py:1085-1271`, `2195-2217` | Catalog: a directory named `x.milk` is listed as a preset; category operations ignore case and pack while the tabs do not; a deleted path stays hidden forever even if re-fetched; restore from quarantine overwrites an existing file — all **VERIFIED** |
| P3-45 | `projectm_effect.py:1379-1410`, `391-412` | Main-thread cost: enabling one preset took 167 ms with 3,000 catalog entries; without the library every construction retries the load (two 26 ms lookups plus five `CDLL` attempts) |
| P3-46 | `projectm_effect.py:304-349`, `770-796`, `1609-1610`, `1798` | Native boundary: no version check; no watchdog and no "preset that was loading" record, so a preset that crashes the renderer is retried; a failed preset's name is shown while the previous one keeps playing; the error path destroys the shared bridge under other instances; library and preset paths from config are unconfined and relative to the working directory.  All 20 foreign functions declare `argtypes` and `restype`, and callbacks are kept referenced |
| P3-47 | `fetch_presets.sh`; `install.sh`; `scan_presets.py` | Scripts (read only): presets are cloned from the moving default branch of three repositories with no pinned commit; `install.sh` does `rm -rf` on the installed copy, wiping its quarantine and trash, on every reinstall; `scan_presets.py` writes a report even with `--dry-run` |
| P3-48 | `tests/test_quarantine.py` | The tests are not hermetic: one builds the real effect (loading the native library with no GL resolver) and was deselected; the other 39 pass but write `runtime/projectm/preset_manager_state.json` into the checkout |
| P3-49 | README; `docs/operations.md`; `HELP_ENTRIES` | Key lists are documented in the README and docs, against the single-source rule; `HELP_ENTRIES` omits E, D, I, Ctrl+A, Enter, Tab and Esc, which core handles.  `textures/` holds 20 committed symlinks with absolute `/home/jj/...` targets |
| P3-50 | `projectm_effect.py` | Eleven `global` statements; `_catalog_lock` defined twice; `random.randrange` uses the global RNG |

---

## What works

- **Robust in normal running.**  68 of 68 effects construct, render, resize
  and destroy cleanly with no GL error and no leaked object.
- **No long-run stall in any game** under music at 30 fps or above.
- **No float32 precision collapse after long travel** in the rollercoast,
  flying or ukiyo-e packs: every hash there already bounds its input, shader
  time is wrapped, and ride coordinates are bounded by the closed circuit.
- **Conventions largely followed:** every effect sets `NAME` and `TAGS`,
  declares parameters, calls `super().update()` first, releases what it
  creates, and reads audio through the normalized fields with a clamp.  No
  `print()`.  No effect ignores audio, reads the wall clock, or writes files.
- **projectM's integration contract matches core:** every member core calls
  exists with the expected shape, including the six that core's capability
  check omits (core audit P3-26).  The app starts and runs without the
  library.  Quarantine and restore validate that paths stay inside the tree.

---

## Status of the 09-30 effects findings in these packs

| 09-30 ID | Status |
|---|---|
| E1 speed changes hard-cut | Changed in core; open in a new form (P2-1) |
| E2 float32 hash collapse | Open in the retro, cosmic, tech, feature and holiday packs; Vector's sparkle is a new member (P3-15).  Not reproduced on this GPU for four of the listed effects |
| E3 time × audio | Open (P2-14); new members Joust, Galaga |
| E4 wrapped phases pop | Open, re-measured, with seven additions (P2-9) |
| E6 Frogger invisible logs | Open, re-verified at 20 % of activations (P3-20) |
| E11 aspect handling | Open |
| E12 Neon Pac ghost reversal | Open, re-verified (P3-21) |
| E13 Mario Kart rivals drift away | Open; not re-run |
| E14 Frogger turtle prediction | Open (P3-20) |
| E15 Particle Storm beat relocation | Open, re-verified (P3-32) |
| E16 Fireworks ignores Candy Frame | Open, wider (P2-19) |
| E19 Sun Ship `seed` | Open (P3-33) |
| E20 ANSI Viewer | Open, re-verified (P3-22) |

---

## What this audit did not do

- No timing on a discrete GPU; no on-screen run; no real music.
- The real projectM effect was never constructed, no preset was loaded or
  timed, and the on-screen consequences of P2-22 and P2-25 were not observed.
- No renders at 21:9 or 4:3 beyond Night Parade and a portrait Galaga
  simulation.
- Mario Kart's rival drift (E13) was not re-run.
- The audience-facing judgment "does this look right" was applied only where
  a defect could be measured.  Taste was out of scope.
- Did not check whether the owner's preset library contains file symlinks
  (P1-1), or whether the effects browser builds a projectM thumbnail (P2-25).
- Did not fix anything.  P2-1 (two constants per effect), P2-2, P2-5, P2-10
  and P2-21 are each a change of a few lines.

Housekeeping: the projectM test run wrote one ignored file into the audit
seat (`runtime/projectm/preset_manager_state.json`, 70 bytes); it was moved
to `/var/tmp/uv-fx-audit/mine/stray/`.
