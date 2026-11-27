# Control Room and multi-head audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/control-room-01` at `ae0df81` (`0.24.0`): `control_room.py`
  (4,151 lines), `install.sh`, README and docs.
- `drop-ins/multi-head-01` at `eda5460` (`1.1.0-rc.3`): `multihead.py`
  (696 lines), plus core's side of it in `unicornviz/app.py` (driver
  selection, `set_display_mode`, mirror rectangles, the calls into the
  controller).

Method:

- Both Python files were read in full.  For the Control Room: `operations.md`,
  `configuration.md` and `integration.md` were read; the other three docs and
  most of the README were searched, not read.
- The Control Room was driven headless: the controller built as its tests
  build it, with a fake `vj_api`, a fake window and the SDL dummy driver,
  synthetic events fed in, and what it drew and called recorded.
- Multi-head was exercised on pure logic with the SDL dummy driver and
  patched display queries.
- **No real window, GL context or display was opened.**  Native behavior
  (destroying a window inside its event handler, HiDPI, the cost of the
  preview readback on the render thread, anything on Wayland) is by reading.
- **VERIFIED** = reproduced by script.  **SUSPECTED** marks what could not be
  confirmed.

Scripts: `/var/tmp/uv-cr-audit/{cr,mine}/`.

Severity: **P1** can crash or freeze the app, corrupt the main window's
rendering, stall the audience output, or fire an unintended show-changing
action.  **P2** the console freezes or misleads while the show continues, a
control acts on the wrong thing, a display mode misbehaves, or a measurable
cost lands on the render thread.  **P3** latent, cosmetic or a rule
violation.

---

## Baseline

| Check | Result |
|---|---|
| Control Room tests | 180 passed |
| Multi-head tests | 39 passed |
| Version sync | Control Room `0.24.0`, multi-head `1.1.0-rc.3`: code, README header and changelog agree |
| Private access | Control Room: no `app._private` access, no `noqa: SLF001`, no `print`, no `global`; fonts through `unicornviz.fonts` |
| `config.toml` | nothing in either drop-in writes it |

---

## Summary

| ID | Sev | Where | One line |
|---|---|---|---|
| P1-1 | P1 | Control Room + core | Two ordinary gestures destroy the Control Room window from inside its own event handler, which the code's own docstring says must never happen (native crash not reproduced) |
| P2-1 | P2 | Control Room | Turning the status HUD on blanks the whole console behind a CLOSE button that does nothing — **VERIFIED** |
| P2-2 | P2 | Control Room | The MODALS → MIDI button opens a selector that cannot select; SELECT can disconnect the controller — **VERIFIED** |
| P2-3 | P2 | Control Room | The frame-skip leaves a stale frame with live hit rectangles from a closed modal or another page — **VERIFIED** |
| P2-4 | P2 | Control Room | One malformed panel from any drop-in freezes the console — **VERIFIED** |
| P2-5 | P2 | Control Room | Key auto-repeat defeats the Esc-Esc close guard and re-fires pause, load and fullscreen — **VERIFIED** |
| P2-6 | P2 | Control Room | Effect-list keys and the wheel act on every page: Down, Down, Enter on LAYOUT switches the effect — **VERIFIED** |
| P2-7 | P2 | Control Room | LAYOUT UP/DOWN do nothing for panels with equal priority, which is the default — **VERIFIED** |
| P2-8 | P2 | core | With `[control_room] enabled = false` (the default), console flashes and modals are drawn on the audience output |
| P2-9 | P2 | both | Drop-in draw callbacks run on the raster thread; the Displays page queries SDL displays off the main thread every tick — **VERIFIED** (thread) |
| P2-10 | P2 | Control Room | Preview: a 4K frame costs 161 ms to downscale on the raster thread; nothing pauses when the window is minimized — **VERIFIED** (raster cost) |
| P2-11 | P2 | multi-head | Excluding a display between two included ones does not free it: the window still covers it — **VERIFIED** |
| P2-12 | P2 | core | The X11 fallback for multi-head only happens at startup; switching to span or mirror at runtime on Wayland has no guard |
| P3-1…P3-22 | P3 | both | See the P3 tables |

---

## P1

### P1-1. The Control Room is destroyed inside its own event handler

`control_room.py:742-750` (`_destroy_window`), `938-942` (`close_now`).

`_destroy_window`'s docstring: "Must NEVER be called from inside an SDL event
handler.  Destroying an SDL window while events for it may still be in flight
causes use-after-free style crashes on X11/Mutter."  `close_now()` is
documented as only for callers known to be outside the handler.

Two ordinary gestures reach it from inside `on_sdl_event`:

- **Shift+M with the console focused.**  `control_room.py:1094-1097` forwards
  the key to core (`app.py:1417` → `hotkeys.py:990`), the controller's own
  `handle_key` (`543-548`) calls `toggle_control_room`, and
  `_destroy_control_room` (`app.py:1662`) calls `close_now()`.
- **Displays → MOVE CONTROL ROOM HERE.**  `control_room.py:1027` → `1415` →
  `3312` → `multihead.py:549-565` → `App.set_control_room_display`
  (`app.py:1705-1706`), which destroys and recreates the controller.

What was established headless (`cr/e3_reentrant.py`): no Python exception;
the old controller ends cleanly; the old teardown does not remove the new
controller's registration; the main GL context is rebound on every path.  The
join of the raster thread cannot deadlock, but it blocks the main loop for
the rest of the current raster frame, up to a 1.0 s timeout (299 ms measured
with a 0.4 s page draw; with a 1.5 s draw the join timed out and the old
thread was still alive).

**Not established:** the native crash.  SDL could not be run here.  The
rating rests on the code's own warning and on the fact that the app uses the
X11 driver whenever it starts in a multi-head mode, which is the tracked
`config.toml`'s setup.  If the docstring's warning is out of date, this is a
P3 contract violation.

**Fix:** have `close_now()` defer when called during `on_sdl_event` (a
re-entrancy flag, with the teardown in `update()`), and have core recreate
the controller on the next frame.

---

## P2

### P2-1. The status HUD blanks the console — VERIFIED

`control_room.py:1510`, `3637-3653`, `1297-1313`; `unicornviz/overlays.py:6807`.
The console treats any non-empty modal snapshot type as a fullscreen modal.
`hud_overlay` has no draw branch and no close branch.  While the HUD is
showing on the audience window (60 s by default, indefinitely with auto-hide
off) the console shows only "HUD OVERLAY (FULLSCREEN)" and a CLOSE button
that calls nothing.  A `webcam_editor` snapshot renders an empty frame the
same way.

**Fix:** ignore snapshot types the console cannot render.

### P2-2. The MIDI launcher cannot select a device — VERIFIED

`control_room.py:1399-1402`, `3722-3731`, `1276-1288`; `overlays.py:6750`,
`6775`.  The launcher opens the MIDI selector and the controller-help modal
together; core's snapshot priority always returns `controller_help`, so the
device list is empty and "Active: (none)" is shown with a port connected.
SELECT at index 2 flashes "MIDI: invalid selection"; at index 0 it calls
`select_midi_device('')`, which disconnects the controller.

**Fix:** a dedicated `vj_api` getter for selector entries.

### P2-3. Stale frames with live hit rectangles — VERIFIED

`control_room.py:3224-3262`, `1510-1514`, `1536`.  The whole-tick skip
compares a fingerprint that the modal branch and the MAIN branch never reset.

- On a panel page, open and close a modal without moving the mouse
  (keyboard, MIDI, or from the audience window): the console keeps showing
  the closed modal with five live `modal_audio_*` hit rectangles until the
  mouse moves.
- With flash routing off (see P2-8), Ctrl+Tab from FX to MAIN and back: the
  page is `fx`, but the published frame and hit rectangles are still MAIN's,
  including PAUSE.

**Fix:** reset the fingerprint whenever another branch publishes a frame.

### P2-4. One malformed panel freezes the console — VERIFIED

`control_room.py:2993` (outside the try at `2981-2990`), `3221`, `3243`,
`3051-3052`, `2973`, `3069`.  Core's panel contract promises an error box for
a bad panel.  Only `content()` raising or returning the wrong type is
isolated.  A `PanelContent` with `rows=None`, a tuple in `rows`,
`columns='2'`, `status=123`, `None` in `meters`, a non-string `accent`, a
string in `pads`, or `title=None` loses every frame: the window freezes on
its last frame with stale hit rectangles, and a warning is logged about 30
times a second (`:831`, unthrottled).  The raster thread itself survives.
NaN, `None` and string meter values are handled.

**Fix:** wrap fingerprinting and body drawing per panel; validate field
types in the dataclasses.

### P2-5. Key auto-repeat — VERIFIED

`control_room.py:1049-1093`.  `event.key.repeat` is not checked for keys the
console handles itself.  Holding Esc for about half a second confirms the
two-press close guard with one physical press.  Five repeat Space events give
five `toggle_pause` calls.  Enter re-fires `goto_effect`; F flips fullscreen
on each repeat.  (The mixer has the same defect: mixer audit U-P2-1.)

**Fix:** ignore repeats for Esc, Space, Enter and F.

### P2-6. List keys act on every page — VERIFIED

`control_room.py:1074-1093`, `996-998`.  On the LAYOUT page, Down, Down,
Enter calls `goto_effect('Effect 5')` with no effect list visible.  Space
pauses from any page.  The wheel scrolls the hidden list from anywhere in the
window.

**Fix:** gate these on the MAIN page and the list's rectangle.

### P2-7. LAYOUT reorder is a no-op by default — VERIFIED

`control_room.py:2683-2695`.  UP and DOWN swap two panels' `order` values.
Equal priority is the default (0), so the swap exchanges identical values and
nothing moves.

**Fix:** renumber the page's panels 0..n-1 before swapping.

### P2-8. Console activity shows on the audience output

`unicornviz/app.py:1739-1749`; `overlays.py:3890-3985`.  The gate that routes
flash messages and modals to the console tests the `[control_room] enabled`
config flag, not whether the window is open.  `enabled` defaults to false
and means "open at startup".  An operator who opens the console with Shift+M
gets every click's flash message, and every modal launched from MODALS, drawn
on the audience output.  It also makes the second case of P2-3 the normal
one.  By reading.

**Fix (core):** gate on `is_open` only.

### P2-9. Drop-in callbacks on the raster thread — VERIFIED (thread)

`control_room.py:2930` (`page.draw`), `2976` (`panel.draw`), `2982` and
`3216` (`content()`), `2522` (`available()`).  All run on the
`ControlRoomRender` thread.

- multi-head's Displays page (`multihead.py:399-409` →
  `all_detected_displays`, `251-254`) therefore calls
  `SDL_GetNumVideoDisplays` and `SDL_GetDisplayBounds` off the main thread on
  every tick the page is shown.  SDL's video API is not documented as
  thread-safe.
- `content()` is called twice per tick on panel pages whenever anything
  changed (8 calls a tick for 4 panels), against the contract's "once per
  frame".

Input is handled correctly: `on_action` and every `vj_api` mutation run on
the main thread.  Slider setters fire once per mouse-move event, unthrottled.

**Fix:** cache display topology on the main thread in `update()`; reuse the
fingerprint pass's content for the draw.

### P2-10. Preview cost — VERIFIED (raster side)

`control_room.py:1771-1784`; `unicornviz/app.py:9744-9785`.

- **Main thread (by reading, not measured):** a synchronous `fbo.read` with
  no PBO of a downsample up to 960 wide, plus a full-canvas framebuffer copy
  and a quad pass, at the console's default cap of 30 a second.
- **Raster thread:** converting and LANCZOS-resizing each capture costs
  15.2 ms for a 960×540 source, and 161 ms when the streaming tap hands over
  a full 3840×2160 frame.
- `preview_fps_cap = 0` removes both costs.
- Nothing pauses when the window is minimized or hidden.

**Fix:** a cheaper filter; request the readback at panel size; stop capture
and raster while minimized.

### P2-11. An excluded middle display is still covered — VERIFIED

`multihead.py:614-622` (`all_display_bounds`), `172-181`; used at
`app.py:8903-8941`.  The span or mirror window is sized to the union of the
included displays.  With three side-by-side 1920-wide displays and the middle
one excluded:

- active displays: 0 and 2;
- window: `(0, 0, 5760, 1080)`, which covers display 1;
- mirror tiles: `(0, 0, …)` and `(3840, 0, …)`.

In span mode the middle display shows the middle of the picture.  In mirror
mode it is covered by the borderless window with no tile drawn: black.  An
exclude only frees a display at an edge of the arrangement.

This is inherent to the single-window design recorded in
`MATE-X11-MULTIHEAD-NOTES.md`; what is missing is any warning.  The Displays
page marks the display "EXCLUDED" and "Monitor excludes saved" is reported.

**Fix:** warn when an exclude is interior to the union; or document that
only edge displays can be excluded.

### P2-12. No driver check when the mode changes at runtime

`unicornviz/app.py:2231-2248`, `8878-8990`.  The app starts on Wayland and
falls back to X11 only when the configured mode at startup is not `single`,
because "Wayland may ignore [window placement] by design".
`set_display_mode()` (the `x` hotkeys and the Displays page's mode buttons)
has no such check and no warning.  A session that starts in `single` on
native Wayland and is switched to span or mirror resizes the window to the
union and asks for a position Wayland ignores; mirror tiles are then computed
from a window origin SDL reports as 0, 0.

By reading; not run on a Wayland session.  The tracked `config.toml` starts
in mirror mode, so that setup takes the X11 fallback at boot and is not
affected.

**Fix:** refuse or warn in `set_display_mode` when the driver is Wayland and
the target mode is not `single`.

---

## P3

### Control Room

| ID | Where | What |
|---|---|---|
| P3-1 | `control_room.py:980-981`, `925` | On the Pillow path the MAIN page raises on every frame at 640×360 and 320×240 — **VERIFIED**.  No minimum window size is set |
| P3-2 | `control_room.py:3247-3253` | Tooltips never appear on static panel pages: the skip fingerprint ignores hover time — **VERIFIED** |
| P3-3 | `control_room.py:848-850`, `3261` | A GPU atlas reset mid-frame on a static page is never re-recorded; the renderer refuses the frame and the hit rectangles stay MAIN's — **VERIFIED** (reset injected) |
| P3-4 | `control_room.py:1773-1774` | Preview bytes and dimensions come from two calls: short bytes lose a frame, long bytes draw garbage.  Not a segfault |
| P3-5 | `control_room.py:2604`, `2614` | A NaN or infinite stored panel order loses every frame — **VERIFIED** |
| P3-6 | `control_room.py:2444`, `470`, `2561` | Starting on a stored `deck_sim` page skips its enter hook: no pad flash or fader mirror until the page is left and re-entered — **VERIFIED** |
| P3-7 | `control_room.py:1674-1686` | With eight page tabs at 960 wide the tabs overlap the title; at `ui_scale` 2.0 the MAIN and Displays tabs are off-screen — **VERIFIED** |
| P3-8 | `control_room.py:1022-1027` | A double-click fires every button twice: RECORD starts then stops; a deck-sim pad sends its note twice — **VERIFIED** |
| P3-9 | `control_room.py:3520` | Deck-sim faders draw at 0 until a real move is seen; the first click jumps the CC to the clicked value.  The grid orientation is correct |
| P3-10 | `control_room.py:742-778`, `2468-2497` | Pages are never unregistered on close: `deck_sim` and `layout` stay bound to the dead controller, and each open re-registers the standard pages over any drop-in page of the same name — **VERIFIED** |
| P3-11 | `control_room.py:732`, `960-994` | The audience display is avoided only in `single` mode; on a one-display machine the default borderless window covers the show; nothing reacts to display hot-plug |
| P3-12 | `control_room.py:2597`, `424` | A stored page override naming a removed page hides the panel on every page (LAYOUT can bring it back); the effect list is read once at construction (SUSPECTED stale) |
| P3-13 | `runtime_state.py:141-160`; `control_room.py:2699`, `2575` | Each stored setting is a synchronous full-file write on the main thread; layout reset writes once per panel; the raster thread does a locked deep-copy read per panel per pass |
| P3-14 | `control_room.py:831`, `2505`, `2590`, `2642` | Raster exceptions are caught per frame with no throttle or backoff.  The known `vj_api.operator_panels()` race (core audit P3-15) costs a frame with no tabs or panels |
| P3-15 | `control_room.py:3694`, `3792-3806`, `954`, `2381`; docs | Key hints are hard-coded on screen and documented in `operations.md`, `troubleshooting.md` and `configuration.md`.  `HELP_ENTRIES` omits Tab, Space, Up/Down and Enter.  The APC map text mentions knobs the mini mk2 does not have |
| P3-16 | `docs/operations.md`, `configuration.md`, README | Stale docs: retired DISPLAY, MONITORS and WEBCAM panels, a single-Esc close, `render_interval = 0.125` (code: 1/30).  Core's `_DEFAULTS['control_room']` lacks `fullscreen`, `render_interval`, `gpu_ui`, `preview_fps_cap` |
| P3-17 | `install.sh` | `--root ""` resolves the `rm -rf` target to `/drop-ins/control-room-01`; no Windows path |

**Frame cost** (1920×1080, 80 effects, 12 panels, preview on):

| Path | MAIN | Panel pages |
|---|---|---|
| GPU draw list | 10 ms mean, 22 ms p99 | 2-3 ms; skipped entirely when static |
| Pillow fallback | 52-65 ms mean | 13-31 ms; the static skip never fired in the harness (cause not investigated) |

The Pillow fallback cannot hold 30 frames a second at 1080p.  The thread
sleeps its full interval after each frame, so its real rate is
1 / (frame time + interval).  On the GPU path the largest costs are the
preview resize (13 ms per capture) and about 78 text measurements of
unchanged strings per frame (4 ms).

### Multi-head

| ID | Where | What |
|---|---|---|
| P3-18 | `multihead.py:183-213`, `113-136`, `594-612` | A display whose bounds query fails is skipped, so list position no longer equals SDL display index: excluding display 2 excluded nothing, and `display_bounds(1)` returned display 2's rectangle — **VERIFIED** |
| P3-19 | `multihead.py:671-696` | After a persistent readback failure, every frame makes a failed asynchronous attempt and then a synchronous read, with one warning for the session — **VERIFIED** |
| P3-20 | `multihead.py:215-234` | `resolve_display_mode()` does not refresh the active layouts: excludes apply only at the next display enumeration — **VERIFIED** in isolation.  Core enumerates again after every mode change, so this is latent |
| P3-21 | `unicornviz/app.py:999-1004`, `1845-1848`, `1812`, `8897-8899` | Core sets the controller's private fields with `setattr` and reads `_active_display_indices`; each mode switch enumerates and logs every display twice |
| P3-22 | `multihead.py:172-181`; `app.py:8950-8963` | Mirror rectangles come from SDL display bounds (logical units) and are used as GL viewports (pixels): on a scaled display the tiles would be wrong (SUSPECTED; not run on a HiDPI rig) |

---

## What works

- **Frame publication is atomic.**  The frame, its size and its hit
  rectangles are published and consumed under one lock, so the past
  segfault pattern (bytes and dimensions out of step) does not apply to the
  Control Room's own frames.
- **Threading of input is right.**  Every action and every `vj_api` mutation
  triggered by console input runs on the main thread.  The raster thread
  makes no GL or SDL call of its own.
- **The main GL context is restored** on every create, present and destroy
  path, including error paths.
- **The raster thread survives any exception** in a frame.
- **Layout persistence** uses the runtime store, writes atomically, and
  writes on click, not on every drag move.
- **Multi-head's staged excludes, save and reset** behave as documented, and
  an exclude list that would remove every display is ignored with a warning.

---

## Status of earlier findings

| Earlier item | Status |
|---|---|
| 09-30 P3-13: `vj_api.operator_panels()` iterates a live dict | Open in core; effect here is one empty frame (P3-14) |
| 09-05 O7: unused locals in `control_room.py` | Fixed |
| 09-05 multi-head: saved monitor excludes were never read | Fixed (`multihead.py:59-73`); applied live since 1.1.0 |
| 2026-07-11 audio degradation (GIL held by a full-window Pillow raster) | Still possible with `gpu_ui = false`; on the GPU path the cost is about 10 ms per MAIN frame |

The 09-30 UI, hardware and Windows audits name nothing in these two
drop-ins.

---

## What this audit did not do

- No real SDL window, GL context, display or compositor.  P1-1's native
  crash, P2-12 on Wayland, P3-22 on HiDPI and the main-thread cost of the
  preview readback were not observed.
- Three Control Room docs and most of its README were searched, not read.
  `unicornviz/deck_sim.py` was not exercised.
- Did not investigate why the Pillow path's static-page skip never fired.
- Did not fix anything.  P2-5, P2-6, P2-7 and P2-8 are each a few lines.
