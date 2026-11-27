# Textures, Sims, Unicorn Tears and Grand Finale audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/textures-01` at `80ea214` (`0.9.0`): Texture Showcase (289
  lines).
- `drop-ins/sims-01` at `f0ce04e` (`0.7.2`): Sim Showcase and its USD loader
  (1,697 lines), plus two tools (512 lines).
- `drop-ins/unicorn-tears-01` at `08df5d2` (`1.0.0-rc.3`): the Unicorn Tears
  effect and controller, dancing unicorn, Rainbow Nova and screen burst
  (1,305 lines).
- `drop-ins/grand-finale-01` at `44e7f1f` (`0.9.0`): the finale sequence and
  its overlay (538 lines).

Method:

- Every line was read, including the shaders.  No sub-agent was used.
- Each piece was run in an offscreen GL context: activation and render
  timing, known test images for the texture wipe and aspect, each sim scene
  rendered with and without depth testing, the dancing unicorn flown start
  to finish at maximum audio, and the finale driven through a full run, an
  abort, and a very long frame against a recording stand-in for `vj_api`.
- The app was not launched.  Timings are on **Mesa Intel Iris Xe**, the only
  GPU on this machine.
- **VERIFIED** = reproduced by script.  **By reading** = found in the code,
  not run.  Scripts are in `/var/tmp/uv-misc-audit/`.

Severity: **P1** = crash, data loss or runaway.  **P2** = wrong picture or
wrong behavior the audience or operator will notice.  **P3** = minor or
hygiene.

## Baseline

| Check | textures-01 | sims-01 | unicorn-tears-01 | grand-finale-01 |
|---|---|---|---|---|
| Own tests | none | 8 passed, 4 skipped | 8 passed | 4 passed |
| ruff | clean | clean | clean | clean |
| bandit (source) | clean | 6 Low | 2 Low | 2 Low |

Bandit, reported and not remediated (all Low):

| ID | Where | Assessment |
|---|---|---|
| B110 try/except/pass ×9 | `sim_showcase.py:169`, `:534`, `:782`; `usd_scene.py:187`, `:369`; `dancing_unicorn_overlay.py:452`, `:459`; `grand_finale.py:397`, `:402` | Deliberate best-effort release and teardown.  `usd_scene.py:187` also hides a failed transform lookup, which would show as a part drawn at the origin |
| B405 XML parsing | `sims-01/tools/fetch_robots.py:19` | The tool parses XML it downloads from a fixed GitHub host.  Low risk; see P3-9 |

## Summary

No P1.

| ID | Sev | Drop-in | Finding |
|---|---|---|---|
| P2-1 | P2 | textures | The wipe runs backwards: the next texture shows first, and every change is a hard cut — VERIFIED |
| P2-2 | P2 | textures | Textures are stretched to the screen shape; the aspect code is never called — VERIFIED |
| P2-3 | P2 | textures | Every activation decodes the whole group on the render thread: about 510 ms, never cached — VERIFIED |
| P2-4 | P2 | sims | Meshes are drawn with no depth test: about half the pixels of each robot are wrong — VERIFIED |
| P2-5 | P2 | unicorn tears | The dancing unicorn's sprite is found only when the app is launched from the repo root — VERIFIED |
| P2-6 | P2 | unicorn tears | The help overlay and README say `U` jumps to the effect; plain `U` replays the splash — by reading |
| P2-7 | P2 | unicorn tears | The effect flashes the whole frame on every beat, outside the flash governor — VERIFIED |
| P2-8 | P2 | grand finale | Abort leaves the hue shift on and wipes the operator's post-FX — VERIFIED |

Plus 19 P3 items.

## P2

### P2-1. The texture wipe runs backwards — VERIFIED

The blend is `smoothstep(wipe − 0.15, wipe + 0.15, uv.x + noise)`
(`texture_showcase.py:120`).  At the start of a cycle (`iMix = 0`) that is
already 1 across most of the screen, so the **next** texture is shown; over
the cycle it wipes back to the **current** one.  When the cycle ends, the
current/next pair advances and the screen jumps to the new "next".

With flat red, green and blue textures and a 2 s cycle:

| Time | Slot A (current) | Slot B (next) | On screen |
|---|---|---|---|
| 0.0 s | green | red | red 99% |
| 1.0 s | green | red | green 52% |
| 1.5 s | green | red | green 90% |
| 2.0 s | red | blue | **blue 99%** (cut from green) |
| 3.5 s | red | blue | red 88% |
| 4.0 s | red | blue | red 99% |

So each change is a hard cut, each texture is wiped in from the one that
follows it, and the order on screen is not the shuffled order.

### P2-2. Textures are stretched — VERIFIED

`cover_uv()` is defined (`:80-92`) and never called; `iResolution` is
declared and unused, so the compiler drops it and the uniform write is
silently skipped (`:266-270`, `:279`).  The texture is sampled with raw
screen coordinates.

A circle in a square texture is drawn 1,213 px wide by 681 px tall on a
1920×1080 target.  The 12 NASA photographs added in 0.9.0 are stretched the
same way.  Pan and zoom also push the sample point past the texture edge,
where it wraps, which is right for a seamless tile and shows a seam on a
photograph.

### P2-3. Every activation reloads the textures — VERIFIED

`_load_textures()` (`:186-225`) decodes and uploads the whole chosen group
in `_init`, with mipmaps, and `destroy()` releases them.  With the shipped
32 textures: 758 ms the first time and **503–518 ms on every later
activation**, on the render thread.  All shipped textures are in one folder,
so "one group" is always all of them.

There is no size cap and no count cap; a `texture_dirs` entry pointing at a
photo folder would decode every file at full resolution each time.  Image
Showcase solved the same problem with a module-level pack cache and a
background prefetch.

### P2-4. Sim meshes are drawn without a depth test — VERIFIED

`render()` (`sim_showcase.py:1113-1138`) draws the triangles without
enabling `DEPTH_TEST`, and core disables it before effects run
(`app.py:8225`).  Triangles therefore cover each other in file order, not by
distance, so far faces show through near ones.  Face culling is off too.

Each mesh scene rendered twice, once as shipped and once with depth testing
switched on externally:

| Scene | Pixels that differ |
|---|---|
| robot_quadruped | 58% |
| robot_hand | 52% |
| robot_hexapod | 52% |
| robot_drone | 49% |
| robot_arm | 48% |
| torus_knot_halo | 28% |
| ocean_waves | 16% |
| cloth_drape | 0% |

Core's render targets do carry a depth buffer (`app.py:3252-3253`), so the
effect only needs to ask for the test.

### P2-5. The dancing unicorn depends on the launch directory — VERIFIED

`_load_texture()` (`dancing_unicorn_overlay.py:225-228`) looks for the
sprite at two **relative** paths.  Core never changes directory.  Launched
from the repo root the sprite loads; launched from anywhere else it does
not, and `render()` then returns before drawing anything, sparkles
included (`:360`).

From another directory: `sprite loaded: False`; one second after a trigger
the overlay reports active and has drawn 0 pixels.  The operator sees the
"UNICORN INCOMING" message and nothing else.  A desktop launcher, an
installed build or a Windows shortcut all start elsewhere.  The effect in
the same drop-in already does it the robust way, relative to its own file
(`unicorn_tears.py:327`).

### P2-6. Help says `U`; the effect is on Shift+U — by reading

`HELP_ENTRIES` lists `U` as "Unicorn Tears effect" (`unicorn_tears.py:304`)
and the README says the same.  `handle_key()` sends plain `U` to
`show_splash()` and returns "Splash replay"; only Shift+U switches to the
effect (`:591-598`).  The splash replay is not in the help overlay at all.

### P2-7. Unicorn Tears flashes the full frame on every beat — VERIFIED

On each beat the shader mixes 42% of a saturated color over the whole frame
and adds 12% more (`unicorn_tears.py:270-274`), decaying over 250 ms.
Measured mean frame level: 80 with no beat, **148 on the beat frame**, back
to 80 after about a quarter second.

This is a full-frame luminance flash at the beat rate, with no rate limit.
The beat-flash drop-in's 3 Hz governor does not cover it (noted as P3-5 in
that audit).  At 180 BPM the beat rate is 3 Hz; any faster beat detection
exceeds it.

### P2-8. Aborting the finale leaves things changed — VERIFIED

`abort()` and normal completion both call `_restore_state()`
(`grand_finale.py:408-415`): auto-advance back, reactivity back, and
`clear_postfx()`.

- **Hue shift stays on.**  The sequence scrolls the hue 4 steps at 2.5 s and
  again at 24 s and 41 s; the only clear is a cue 6 s into the outro
  (`:519`).  An abort at 36 s had applied 8 steps and made three calls, none
  of them `clear_hue_shift`.
- **The operator's post-FX is wiped, not restored.**  Whatever slot was on
  before the finale is cleared on abort and on completion; it is never
  saved.  By reading.
- **The effect is not restored** after the peak's random swap, and a dancing
  unicorn or burst in flight carries on.  By reading.

## P3

### textures-01

| ID | Where | Finding |
|---|---|---|
| P3-1 | `texture_showcase.py:194-203` | No textures found: a full magenta screen (center pixel 254, 0, 251) — VERIFIED |
| P3-2 | `:266-270` | `render()` builds a closure and a tuple every frame and swallows missing uniforms, which is what hid P2-2 |
| P3-3 | drop-in root | No tests; no start-up randomization beyond the order (no random pan phase, zoom or tint) |

### sims-01

| ID | Where | Finding |
|---|---|---|
| P3-4 | `sim_showcase.py:1114-1121`, `:1143-1146` | A vertex array is created and released every frame, along with about twenty small numpy arrays and tuples.  Measured cost is small (0.23 ms CPU, 2.1 ms total at 1080p) — VERIFIED; it is the project's no-allocation rule that is broken, not the frame budget |
| P3-5 | `:1159-1161` | `destroy()` releases the GL buffers of scenes held in the module-level shared cache.  The next activation re-uploads them (5 ms) — VERIFIED; a second live instance would be left with nothing to draw |
| P3-6 | `:46`, `:464`, `:643`, `:935` | Uses the global `random.shuffle` for style order and motion picks.  Project rule: `self.rng` only |
| P3-7 | `usd_scene.py:329-334` | Two simultaneous loads of the same file both do the full work — VERIFIED.  The startup prewarm and a first activation can overlap this way |
| P3-8 | `sim_showcase.py:1142` | `PROGRAM_POINT_SIZE` is enabled and left on |
| P3-9 | `tools/fetch_robots.py:25-43` | A second network tool, with a `# nosec` annotation on the download.  Project rules name `tools/fetch_acid_ans.py` as the only network tool and reserve `# nosec` for owner-approved cases.  The host is fixed and the output folder is git-ignored |
| P3-10 | `tests/` | 4 of 12 tests are skipped in this environment |

Load and render cost are both fine: all 20 scenes parse in 0.4 s cold, the
whole CPU cache is 1.4 MB, and no scene takes more than 2.2 ms to draw.

### unicorn-tears-01

| ID | Where | Finding |
|---|---|---|
| P3-11 | `unicorn_tears.py:461-489`, `dancing_unicorn_overlay.py:172-175` | The "Max sparkles" setting (300/600/900/1,800) cannot change anything: at maximum audio the live count peaks at 208, below the lowest choice — VERIFIED |
| P3-12 | `unicorn_tears.py:311-337` | Each activation reads, hashes and decodes the 2.7 MB avatar image on the render thread: 92–123 ms — VERIFIED.  The hash is stored XOR-obfuscated; a replaced avatar is disabled with a warning |
| P3-13 | `unicorn_tears.py` shader | 9.2 ms per frame at 1080p on this GPU — VERIFIED.  The project guideline is 8 ms on a GTX 1060, which is faster than this GPU, so this is a note, not a breach |
| P3-14 | `rainbow_nova.py:4`, `:130` | Docstring says a 3-second burst; the duration is 1.0 s |
| P3-15 | `dancing_unicorn_overlay.py:380-383`, `:413-442`; `rainbow_nova.py:180` | Render leaves blending (and point-size mode) enabled; the sparkle upload copies with `tobytes()` and two debug log calls run every frame |
| P3-16 | `screen_burst_controller.py:18` | `duration = 0` becomes 0.60; a negative duration makes the burst never fire |

### grand-finale-01

| ID | Where | Finding |
|---|---|---|
| P3-17 | `grand_finale.py:258-262` | `intensity = 0` becomes 1.0 — VERIFIED.  `blackout_s` below 15 is raised to 15 (documented) |
| P3-18 | `:14-16` | Module docstring lists the hotkeys and says they are registered in `overlays.py`; they come from `HELP_ENTRIES` |
| P3-19 | `:319-322`, `:439-444` | The Control Room TRIGGER button calls `trigger()` directly, skipping the flash message the hotkey path shows.  The blackout starts and ends as a hard cut: the scene is fully visible for the last 10 s of the outro, then black for 15 s, then back at once |

## Recommended tests

Each is written to fail against the current code where a finding is open.

### textures-01 (no tests today)

| Test | Covers | Kind |
|---|---|---|
| With three flat-color textures, the screen shows the current texture at the start of a cycle and the next one at the end, and the frame before and after an advance match | P2-1 | GL |
| A circle in a square texture is drawn round on a 16:9 target | P2-2 | GL |
| Every uniform the effect sets exists in the compiled program | P2-2, P3-2 | GL |
| A second activation decodes no image files | P2-3 | Python (fake context, patched `Image.open`) |
| An empty folder renders black, or the effect reports itself unavailable | P3-1 | GL |

### sims-01

| Test | Covers | Kind |
|---|---|---|
| Two overlapping triangles, the nearer one submitted first, show the nearer one | P2-4 | GL |
| `render()` creates no vertex arrays after the first frame | P3-4 | Python (counting context) |
| Destroying one instance leaves a second instance able to draw | P3-5 | Python (fake context) |
| Two simultaneous `load_scene()` calls for one path parse the file once | P3-7 | Python |
| Blend, depth and point-size state after `render()` match the state before it | P3-8, P2-4 | Python (fake context) |

### unicorn-tears-01

| Test | Covers | Kind |
|---|---|---|
| The dancing unicorn's sprite loads with the working directory set to a temp folder | P2-5 | Python (fake context) |
| With no sprite available, a trigger still draws the sparkle trail, or reports failure to the caller | P2-5 | Python |
| The key listed in `HELP_ENTRIES` for the effect is the key `handle_key()` uses for it | P2-6 | Python |
| The effect's per-beat frame-level change stays under a stated bound, or beats closer than a third of a second do not re-flash | P2-7 | GL |
| At maximum audio the live sparkle count reaches the lowest selectable cap (or the choices are lowered to match) | P3-11 | Python |
| A second activation does not re-read the avatar file | P3-12 | Python |

### grand-finale-01

| Test | Covers | Kind |
|---|---|---|
| Abort at any point after the first hue scroll calls `clear_hue_shift` | P2-8 | Python |
| The post-FX slot active before `trigger()` is active again after abort and after completion | P2-8 | Python |
| `intensity = 0` yields an overlay phase of 0 throughout | P3-17 | Python |
| A full run makes each cue's call exactly once, and a single very long frame still fires every cue's restore | holds today for the count; unpinned | Python |
| The Control Room trigger and the hotkey produce the same operator message | P3-19 | Python |

## What works

- **Sim loading and drawing.**  0.4 s to parse all 20 scenes cold, a warm
  activation in 5 ms, 2 ms per frame, and a background prewarm at startup.
- **Finale sequencing.**  Phases start at 0, 8, 28, 43, 49 and 65 s and the
  run ends at 80 s exactly; every cue fires once; a 500-second frame
  advances one phase per frame without skipping the restore; aborting during
  the blackout returns the picture at once.  The overlay costs 2.7 ms.
- **Dancing unicorn.**  A full flight is 5.6 s at 0.5 ms per frame, and it
  cleans up after itself.
- **Texture teardown.**  Texture Showcase releases everything it loads.
- **Independence.**  The finale's calls all go through `vj_api`, and the
  drop-ins load without each other.

## What this audit did not do

- Launch the app or drive the hotkeys, MIDI actions or Control Room panels
  through real input.
- Run the two sims tools; `fetch_robots.py` needs the network.
- Test character packs with skeletal animation or blendshapes: none are
  shipped, so that loader path was read but not run.
- Test Windows or a discrete GPU.
- Judge the look of any scene, texture or sequence, or any tuning value.
- Fix anything.  The bandit findings are reported only.
