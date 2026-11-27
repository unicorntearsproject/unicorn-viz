# Beat-flash, color-grade and candy-frame audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/beat-flash-01` at `a15159b` (`0.10.0`): controller, safety
  governor and config validator (509 lines).
- `drop-ins/color-grade-01` at `11c16ef` (`0.10.0`): controller, grade/LUT
  definitions and config validator (587 lines).
- `drop-ins/candy-frame-01` at `16855e9` (`1.1.0-rc.3`): controller, pack
  base and the eight pattern packs (about 1,020 lines).
- Core's side of all three in `unicornviz/app.py`: loading, the post chain,
  the Candy Frame viewport and the overlay step.

Method:

- Every line of the three drop-ins was read, including the shaders.  No
  sub-agent was used.
- Each was run in an offscreen GL context against known source images: the
  flash governor at 24, 60, 144 and 1000 fps; every flash mode at several
  brightness settings; generated `.cube` files (identity, channel swap,
  non-default domain, malformed, 65³); the frame at several insets and at a
  half-size render target; cost at 1080p and 4K; create/destroy cycles.
- Config handling was checked through core's real `Config.validate()` on
  scratch config files.
- No window was opened and the app was not launched.  Timings are on **Mesa
  Intel Iris Xe**, the only GPU on this machine.
- **VERIFIED** = reproduced by script.  **By reading** = found in the code,
  not run.  Scripts are in `/var/tmp/uv-fx2-audit/`.

Severity: **P1** = crash, data loss or a safety limit that does not hold.
**P2** = wrong behavior the audience or operator will notice, or a document
that promises something the code does not do.  **P3** = minor, cosmetic or
hygiene.

## Baseline

| Check | beat-flash-01 | color-grade-01 | candy-frame-01 |
|---|---|---|---|
| Own tests | 18 passed | 17 passed | 25 passed |
| ruff | clean | 1 (`F541` in `tests/test_grades.py:19`) | clean |
| bandit (source) | 1 Low | 1 Low | 2 Low |

Bandit findings, reported and not remediated:

| ID | Where | Severity | Assessment |
|---|---|---|---|
| B110 try/except/pass | `beat_flash_controller.py:335` | Low | Deliberate: Control Room panel unregister during teardown |
| B110 try/except/pass | `color_grade_controller.py:415` | Low | Same |
| B110 try/except/pass | `candy_frame_controller.py:571` | Low | Same |
| B311 `random` | `candy_frame_controller.py:316` | Low | False positive: shuffles the visual pack order, nothing security-related |

## Summary

No P1.  The one safety mechanism in scope, the 3 Hz flash-rate cap, holds at
every frame rate tested.

| ID | Sev | Drop-in | Finding |
|---|---|---|---|
| P2-1 | P2 | beat-flash | The brightness cap does not stop a full white-out — VERIFIED |
| P2-2 | P2 | beat-flash | `max_hz` above 3 stops the app from starting; the docs say it is clamped — VERIFIED |
| P2-3 | P2 | candy-frame | Same opening pattern and same pattern order every session — VERIFIED |
| P2-4 | P2 | candy-frame + core | Black gap between frame and picture when render scale is below 1.0 — VERIFIED (simulated) |

Plus 27 P3 items in the tables below.

## P2

### P2-1. The brightness cap does not stop a full white-out — VERIFIED

`governor.py:10-11` says brightness is hard-capped "so a flash can never
fully white-out the frame."  The cap limits the governor's level (0.8
maximum, 0.78 default), but the shader then adds to it.

- **Full mode** (`beat_flash_controller.py:99-104`) screen-blends the flash
  and then adds a further `uColor * level * 0.18`.
- **Pulse mode** (`:95-97`) multiplies the picture by `1 + level` and adds
  `level * 0.25`.

Measured output for a flat gray source at the peak of a flash (8-bit values):

| Source | Full, 0.78 (default) | Full, 0.80 | Pulse, 0.78 | Edge, 0.78 (center / corner) |
|---|---|---|---|---|
| 0 (black) | 235 | 241 | 50 | 0 / 193 |
| 64 | 249 | 254 | 164 | 64 / 208 |
| 92 | **255** | **255** | 213 | 92 / 215 |
| 128 | **255** | **255** | **255** | 128 / 224 |
| 191 | **255** | **255** | **255** | 191 / 239 |

At the default setting, full mode drives every pixel brighter than about 36%
to pure white and lifts pure black to 92%.  Edge mode behaves as documented.
The rate cap is what the safety standard depends on and it holds; this is the
secondary limit not doing what its documentation says.

### P2-2. `max_hz` above 3 stops the app from starting — VERIFIED

`config_validator.py:35-39` returns a message for `max_hz > 3.0` that ends
"(it will be clamped to 3.0)".  Core treats every validator message as fatal
(`unicornviz/__main__.py:660-663` exits with status 2).

Run through `Config.validate()`:

```
[beat_flash] max_hz = 5.0  ->  Configuration validation failed:
  dropin:beat-flash-01: [beat_flash] max_hz exceeds the 3 Hz
  photosensitivity safety cap (it will be clamped to 3.0)
```

`docs/configuration.md:22-23` says the validator "warns", and
`docs/troubleshooting.md:20` has a section titled "I set max_hz above 3 but
it isn't faster", a state the operator can never reach because the app does
not start.  The governor would have clamped the value safely on its own.

### P2-3. Same opening pattern and same order every session — VERIFIED

`candy_frame_controller.py:316` seeds the shuffle with
`random.Random(int(cfg.get('seed', 0) or 0))`.  With no `seed` in the config
(the shipped state) the seed is always 0, so the shuffle at `:339-340`,
commented "Shuffle startup to avoid the same opener every session", produces
the same result every time.

Five fresh controllers all opened on **Plasma Ribbon**, and the order is
always Plasma Ribbon, Lava Ooze, Prism Pulse, Neon Weave, Laser Sweep, Paint
Drip, Wax Melt, Scanline Comb.  There is no config value that means "random";
any `seed` just picks a different fixed order.  This is the project's
"visually distinct each time" rule not being met.

### P2-4. Black gap when render scale is below 1.0 — VERIFIED (simulated)

Core asks the frame for the picture's inset rectangle in **render-target**
pixels (`app.py:8168-8175`, called with `_render_width/_render_height`), and
`content_viewport()` answers with the inset in raw pixels
(`candy_frame_controller.py:449-458`).  The frame itself is then drawn at
**window** resolution (`app.py:8400-8402`, `iResolution` = window size).

At render scale 0.5 on a 1920×1080 window, the picture is inset 76 px in a
960×540 target (7.9% of the width) while the frame occupies 76 px of 1920
(4.0%).  Simulated with a white picture in a half-size target: the frame ends
at x = 76 and the picture starts at x = 153, leaving a 77 px black ring all
the way round.  The gap is inside the frame, so the outer-fill feature does
not cover it.  The gap width is `inset × (1/scale − 1)` window pixels.

Render scale is an operator setting (Performance tab, 0.5 to 1.0).  This was
not run in the live app.

## P3

### beat-flash-01

| ID | Where | Finding |
|---|---|---|
| P3-1 | `governor.py:103-106` | Beats dropped by the rate cooldown still advance the division counter.  With `beat_division = 2` and 6 beats per second the result is 1.5 flashes per second, not 3 — VERIFIED.  Errs on the safe side |
| P3-2 | `config_validator.py:27-32` | Booleans pass as numbers (`max_hz = true` runs at 1 Hz).  `max_brightness` above 0.8, `beat_division` of 0 or less, a negative `flash_duration` and a malformed `color` all pass silently, unlike `max_hz` — VERIFIED |
| P3-3 | `beat_flash_controller.py:152-162` | Flash color is unrestricted.  A saturated red flash, which the standards treat more strictly than a general flash, is accepted with the same limits |
| P3-4 | `beat_flash_controller.py:3` | Module docstring says the pass is "drawn last"; video-postfx runs after it (`app.py:2877-2882`) |
| P3-5 | `governor.py:1-5` | The governor limits this pass only.  Any other effect or overlay that flashes the whole frame is outside it |
| P3-6 | `beat_flash_controller.py:336-341` | `destroy()` releases three GL objects inside one `try`; a failure on the first leaves the others unreleased |
| P3-7 | `README.md:28-31`, `docs/operations.md:21-23` | Hotkeys are listed in the README and docs.  Project rule: `HELP_ENTRIES` is the only place keys are documented |

### color-grade-01

| ID | Where | Finding |
|---|---|---|
| P3-8 | `color_grade_controller.py:178-181` | The default `assets/luts` folder is found by walking up from the source file, not from the app root, and a configured `lut_dir` may point anywhere on disk (the validator accepts `/etc`).  Project rule: config paths stay inside the project or an allowed folder |
| P3-9 | `grades.py:108-109` | `DOMAIN_MIN` / `DOMAIN_MAX` are ignored.  A LUT with a domain other than 0..1 is applied as if it were 0..1 — VERIFIED.  The docstring says so; the docs do not |
| P3-10 | `color_grade_controller.py:184-196`, `grades.py:96-115` | LUTs are parsed line by line on the main thread at startup with no size limit.  One 65³ file takes 0.40 s — VERIFIED; a folder of them adds seconds to launch |
| P3-11 | `color_grade_controller.py:325-331`, `:250` | Names are matched to the first hit, case-insensitive, so a LUT file named like a built-in look cannot be selected by name, and one named `Neutral.cube` is never applied — VERIFIED with `neutral.cube` |
| P3-12 | `color_grade_controller.py:202-209` | A misspelled `default_grade` silently becomes a random look, with no log line — VERIFIED |
| P3-13 | `color_grade_controller.py:187-194`, `:358-361` | A LUT joins the list before its texture is created.  If texture creation fails, selecting that look clears the output and returns `False`, and core copies the cleared buffer to the screen regardless (`app.py:2896-2906` ignores the return value): a black frame.  By reading; not reproducible on this GPU with valid files |
| P3-14 | `config_validator.py:20-25` | `intensity = true` passes validation and becomes 1.0 — VERIFIED |
| P3-15 | `color_grade_controller.py:416-424` | Same single-`try` teardown as P3-6.  With one failing release, all five LUT textures and the shader program stayed allocated — VERIFIED |
| P3-16 | `README.md:17-21`, `docs/operations.md:21-23` | Hotkeys listed outside `HELP_ENTRIES` (same rule as P3-7) |
| P3-17 | `tests/test_grades.py:19` | ruff `F541`, f-string without placeholders |

### candy-frame-01

| ID | Where | Finding |
|---|---|---|
| P3-18 | `candy_frame_controller.py:300-307` | The `float(x or default)` pattern turns a configured 0 into the default: `intensity = 0` becomes 1.0, `outer_fill_mix = 0` becomes 0.84, `inset_px = 0` becomes 76 — VERIFIED |
| P3-19 | `candy_frame_controller.py:170-172` vs `:453-455` | The shader and `content_viewport()` clamp the inset differently.  `inset_px = 4`: the frame is drawn 12 px wide over an 8 px strip of picture.  `inset_px = 600` at 1080p: the frame stops at 486 px and the picture starts at 538 px — VERIFIED |
| P3-20 | drop-in root | No `config_validator.py`.  `inset_px = "wide"` passes startup validation, then the controller raises and Candy Frame is silently absent (one warning in the log).  A negative `intensity` or `inset_px` and an unknown `outer_fill_mode` are accepted — VERIFIED |
| P3-21 | `effects/base.py:50-69`, `candy_frame_controller.py:323-337` | One pattern file that fails to import takes the whole drop-in down — VERIFIED.  Only pack construction is guarded, not the import.  The shader program and buffers created just before discovery are not released on that path (by reading) |
| P3-22 | `README.md` | Four statements do not match the code: default `inset_px` is 76, not 96; the frame is described as rounded but the corner radius is fixed at 0 (`:529`); the documented per-pack `[candy_frame.effects."<name>"]` overrides are read by no pack; the hotkey is listed outside `HELP_ENTRIES` |
| P3-23 | `candy_frame_controller.py:489-492`, shader | NaN or infinite audio levels turn the whole frame band black until the values recover — VERIFIED.  No crash |
| P3-24 | `candy_frame_controller.py:555-558` | `render()` enables blending and leaves it enabled for whatever draws next (by reading).  Core resets GL state only before the HUD |
| P3-25 | `candy_frame_controller.py:53-285` | The fragment shader is 232 lines against the project's 150-line limit and has no header comment naming its uniforms.  Cost at 4K on this GPU: 1.7 ms classic, 5.7 ms with a drip pack and outer fill — VERIFIED.  At 1080p: 0.8 and 1.5 ms |
| P3-26 | `candy_frame_controller.py:573-575` | `destroy()` releases are not individually guarded (core does contain the failure, `app.py:8123-8128`) |

### Core side

| ID | Where | Finding |
|---|---|---|
| P3-27 | `app.py:8239-8242` vs `:8267-8269`, `:8162`, `:8393` | With the projectM manager open, `_candy_active()` reports the frame as off, so the scene may be drawn straight to the screen, but the viewport still shrinks the effect and the overlay step still composites the frame from `fbo_a`, which was not updated that frame.  By reading; not run |

## What works

- **Flash rate cap.**  With a beat edge every other frame for ten seconds, no
  one-second window held more than 3 flashes at 24, 60, 144 or 1000 fps, and
  a configured `max_hz` of 99 behaved identically.  Core passes unscaled
  frame time, so playback-speed controls cannot raise the rate.
- **Flash safety defaults.**  Off at boot; a NaN beat or NaN frame time arms
  nothing and breaks nothing; NaN brightness resolves to 0; disabling resets
  the envelope.
- **Color-grade accuracy.**  An identity LUT and the procedural neutral look
  are both exact to the 8-bit value; a channel-swap LUT confirms the axis
  order is right; intensity mixes linearly; the Neutral look skips the pass
  entirely; malformed and 1D `.cube` files are rejected with a clear warning.
- **Color-grade cost.**  0.35 ms per pass at 1080p, procedural or 65³ LUT.
- **Candy Frame picture area.**  Inside the frame, output equals input on
  every pixel at the default inset, with and without outer fill.
- **Candy Frame timing.**  Patterns cycle on schedule, a one-off huge frame
  time causes one cycle rather than a burst, and the frame freezes cleanly
  while hidden.
- **Lifecycle.**  200 create/destroy cycles of the beat-flash controller left
  no GL error.  Core loads all three through guarded loaders and runs without
  any of them.
- **Earlier finding closed.**  The 2026-09-30 P2-4 (Candy Frame hiding
  transitions) is fixed in the current core code (`app.py:8608-8621`), by
  reading.

## What this audit did not do

- Launch the app, open a window, or drive the hotkeys, MIDI actions or
  Control Room panels through real input.
- Measure flashes against a formal photosensitivity analysis tool; the rate
  figures are counts of governor arm events and the brightness figures are
  read-back pixel values.
- Test on Windows or on a discrete GPU.
- Judge the look of any grade or pattern, or any tuning value.
- Fix anything.  The ruff and bandit findings are reported only.
