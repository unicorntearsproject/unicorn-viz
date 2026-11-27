# Video Out and MIDI Controllers audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/video-out-01` at `54d1ba7` (`0.7.1`): the controller, the
  v4l2loopback sink, the PipeWire sink (1,122 lines) and two shell scripts.
- `drop-ins/midi-controllers-01` at `9af44c1` (`0.11.1`): the MIDI manager
  and its modals, the APC mini mk2 LED/USB driver, presets and profiles
  (3,929 lines).
- Core's side of both: the frame tap, the drop-in loader, MIDI dispatch.

Method:

- video-out was read directly.  The four MIDI files were read by one
  read-only sub-agent; its scripts were then inspected and rerun by hand,
  and the code behind its top findings was read directly.
- **No hardware was touched.**  An APC mini mk2 and a DDJ-REV1 were plugged
  in during the audit.  Every MIDI script replaces `rtmidi` with a stand-in
  that raises on any port access, replaces the libusb handle with a fake,
  and disables the raw-MIDI helpers before importing the drop-in.  No MIDI
  or USB message was sent and no device was claimed.
- video-out was run against stand-in file descriptors.  No real video
  device was opened, and there is no v4l2loopback node or libfunnel build on
  this machine, so neither backend was run against a real consumer.
- The app was not launched.
- **VERIFIED** = reproduced by script, with fake hardware where hardware is
  involved.  **By reading** = found in the code, not run.  **SUSPECTED** =
  plausible, needs the device to confirm.  Scripts are in
  `/var/tmp/uv-midi-audit/` and `/var/tmp/uv-misc-audit/` (`t12.py`,
  `t13.py`).

Severity: **P1** = crash, data loss, a hang, or a runaway or dead hardware
state that needs a restart.  **P2** = wrong behavior the operator will
notice, or a broken rule with real consequences.  **P3** = minor or hygiene.

## Baseline

| Check | video-out-01 | midi-controllers-01 |
|---|---|---|
| Own tests | 37 passed | 9 passed (two test files for 3,929 lines) |
| ruff | clean | clean |
| bandit (source) | 4 Low | 14 Low |

Bandit, reported and not remediated (all Low): 15 × `B110` try/except/pass
(11 in `midi_manager.py`, 3 in `pipewire_sink.py`, 1 in
`video_out_controller.py`), 1 × `B112` in `midi_manager.py`, and 2 × `B105`
"hardcoded password" in `apc_leds.py` and `controller_presets.py`, which are
false positives on ordinary strings.  Several of the `B110` sites in
`midi_manager.py` hide failures that feed findings below (failed sends,
failed registrations).

## Summary

| ID | Sev | Drop-in | Finding |
|---|---|---|---|
| P1-1 | P1 | midi | After the APC is unplugged, the USB reader thread spins one CPU core until exit — VERIFIED (fake USB) |
| P1-2 | P1 | midi | Nothing reconnects the APC on the USB path: pads, faders and LEDs stay dead until restart — partly VERIFIED |
| P2-1 | P2 | midi | `close()` frees the USB device while the writer may still be mid-transfer — VERIFIED (order), outcome SUSPECTED |
| P2-2 | P2 | midi | MIDI Learn binds on the device thread and calls `vj_api` from it — VERIFIED |
| P2-3 | P2 | midi | The pad press that teaches a binding also fires the new action — VERIFIED |
| P2-4 | P2 | midi | Learn accepts messages from other devices and never times out — VERIFIED |
| P2-5 | P2 | midi | Unbound pads stay lit — VERIFIED |
| P2-6 | P2 | midi | Failed LED sends are recorded as sent; an APC plugged in after launch never lights — VERIFIED |
| P2-7 | P2 | midi | The APC is claimed whatever `[midi]` says — by reading |
| P2-8 | P2 | midi | The raw-MIDI fallback matches any sound card with "mk2" in its name — VERIFIED (matcher) |
| P2-9 | P2 | midi | One profile file that is not UTF-8 disables every profile — VERIFIED |
| P2-10 | P2 | midi | Saved profiles are not complete: cleared pads return, and the second port and device scope are lost — VERIFIED |
| P2-11 | P2 | midi | Profile name collisions lose saves silently; Enter in Save-as overwrites the stock profile's key — VERIFIED |
| P2-12 | P2 | midi | An open MIDI modal costs 37 ms per frame at 1080p — VERIFIED (CPU side) |
| P2-13 | P2 | video-out | The PipeWire backend cannot load under core's loader — VERIFIED |
| P2-14 | P2 | video-out | The first toggle reports "on" when nothing is publishing — VERIFIED |
| P2-15 | P2 | video-out | The virtual-camera picture is upside down — by reading |

Plus 24 P3 items.

## P1

### P1-1. The USB reader spins a core after unplug — VERIFIED (fake USB)

`_read_loop()` (`apc_leds.py:660-673`) calls the bulk-IN transfer in a
`while` loop and acts only on a return code of 0.  Any other code loops
straight back with no sleep, no exit and no reconnect.  The 10 ms timeout
paces the loop only while the device is present; once it is unplugged
libusb returns "no device" immediately.

With a fake libusb that returns that code: **1,653,682 calls in 0.5 s, 0.50 s
of CPU in 0.51 s of wall time**.  On the real path the call releases the
interpreter lock, so the cost is one busy core plus lock churn against the
render thread, for the rest of the session.

### P1-2. Nothing reconnects the APC on the USB path — partly VERIFIED

- The writer treats every error as a timeout: after three it pauses LED
  output for 30 s, then probes the same dead handle again
  (`apc_leds.py:591-627`).  With the fake: 3 writes, paused 30 s, 2 pushes
  dropped, and no method anywhere that reopens or re-claims the device —
  VERIFIED.
- Claiming the device at startup detaches the kernel driver, which removes
  the APC's ALSA ports (`:757-765`).  After a replug, core's own MIDI start
  therefore finds nothing to open either.  By reading.

The net effect of a knocked cable during a set: the APC is dead until the
app is restarted, with P1-1 running the whole time.  The real unplug
sequence needs the device to confirm.

## P2

### midi-controllers-01

**P2-1. `close()` frees the device under a running transfer.**
`_stop_writer` joins for 1.0 s and `close()` then releases the interface,
closes the handle and exits libusb regardless (`apc_leds.py:566-572`,
`:646-658`).  With a transfer still in flight the recorded order was
`transfer-start, release, close, exit` with no `transfer-end` first.
Whether the real library crashes or hangs there is SUSPECTED; the reader
and the raw-MIDI reader follow the same pattern with shorter joins.

**P2-2. Learn runs on the device thread.**  `_on_midi`
(`midi_manager.py:1619-1628`) is a named listener, so core calls it on the
rtmidi thread (or the APC reader thread).  It then calls `midi_bind_note`,
`midi_unbind_note` and the CC equivalents directly, changing the note and CC
maps with no lock while the render thread reads them.  The project rule is
that the callback may only queue.  Observed: `midi_bind_note ran on thread:
FAKE-rtmidi-thread`.  It also logs the bound note at INFO from that thread,
which the rules restrict to DEBUG.

**P2-3. Teaching a pad fires the action.**  Core queues the same note-on
that Learn just consumed, and the render thread resolves it after the bind
has landed (`midi_manager.py:684`, `:699`; `hotkeys.py:497-527`).  Learning
`postfx_clear` fired `postfx_clear`.  Learning `context_select` synthesizes
Return, which re-arms the modal, so the next pad pressed steals the binding;
learning `context_back` synthesizes Escape and closes it.

**P2-4. Learn accepts any device and never times out.**  There is no check
on the event's source (`:1625-1628`): with the CC tab armed, a CC from the
DJ mixer was bound (`{33: 'speed'}`, replacing `{74: 'speed'}`).  Learning
one pad for an action also removes every other pad bound to that action.
Once armed, Learn waits indefinitely.

**P2-5. Unbound pads stay lit.**  The refresh only queues notes that are
currently bound (`apc_leds.py:1274-1283`), and a profile switch clears the
"sent" record without sending an off (`:1139-1144`).  After unbinding a
pad no message was sent and its last color stayed 33; switching to a
one-pad profile sent one message.  The module docstring says unbound pads
go dark.

**P2-6. Failed sends count as sent.**  `_sent[note]` is set whether or not
the send succeeded, and the host-mode flag is set unconditionally
(`apc_leds.py:1306-1311`, `:1336`).  Started with no APC and then given
one: no SysEx and no colors were ever sent, while the driver believed three
pads were lit.  The module docstring promises an automatic resend.

**P2-7. The APC is claimed regardless of config.**  Preset registration
opens the USB path for any plugged APC (`controller_presets.py:447-457`,
`apc_leds.py:750-765`), detaching its kernel driver and injecting its
events into dispatch, even with `device = ""` or "no device" chosen in the
selector.  If the last registration step fails, the interface stays claimed
with no reader.  By reading; the injection half was reproduced.

**P2-8. The raw-MIDI fallback matches the wrong device.**  The card matcher
accepts any card whose name contains "mk2" (`apc_leds.py:176`):
`Launchkey MK2 49` and `MiniLab mk2 synth` both matched.  The matched port
is then sent the APC's SysEx and pad colors.  Matcher VERIFIED with a
mocked card list; the effect on such a device is by reading.

**P2-9. One bad file disables all profiles.**  `_load_one()` catches
`OSError` only (`controller_profiles.py:388-397`); a file that is not valid
UTF-8 raises `UnicodeDecodeError` out of `discover()`, and startup then
registers no file profiles at all.

**P2-10. Saved profiles are not complete.**  Core layers generic defaults
under every applied profile, and Save-as captures them: five notes and six
CCs were live that the profile file never named, and a pad cleared before
saving came back as `next` when the saved profile was applied.  A saved
profile also re-registers one port where the stock preset opens two
(`[1, 0]` vs `[0]`), and its device field is saved empty because the USB
path has removed the port names it reads.

**P2-11. Name collisions lose saves.**  Profiles are keyed by their name
but the file that sorts last wins (`controller_profiles.py:371-376`,
`:419`): a save of note 99 was shadowed by `zz-backup.toml`.  `Set A!` and
`Set A?` both write `set-a.toml`.  The Save-as box is pre-filled with the
active key, so Ctrl+S then Enter writes `apc-mini-mk2-stock.toml` and
shadows the bundled stock profile; the store's `delete()` has no caller, so
there is no undo in the app.

**P2-12. Modal frame cost.**  Each frame with a modal open builds a
full-window image with about 760 gradient lines, converts it and uploads it
(`midi_manager.py:707-843`, `:1135-1159`).

| Modal | 1920×1080 | 3840×2160 |
|---|---|---|
| MIDI Learn | 36.6 ms, 8 MB upload | 65.2 ms, 33 MB upload |
| Profile selector | 13.0 ms, 8 MB upload | 29.4 ms, 33 MB upload |

CPU side only; the GPU upload comes on top.

### video-out-01

**P2-13. The PipeWire backend cannot load.**  `submit_texture()` imports
its sibling with `from pipewire_sink import PipeWireSink`
(`video_out_controller.py:599`).  Core loads drop-ins by file path and does
not put the drop-in's folder on the import path.  Loaded through core's own
`load_dropin_symbol`:

```
video-out: PipeWire backend unavailable: No module named 'pipewire_sink'
```

The drop-in's tests add the folder to the path themselves, which is why
they pass.  The backend's own docstring says its pixels are unproven; under
core it never gets as far as trying.

**P2-14. The first toggle reports "on" regardless.**  `toggle()` opens the
sink at once only when the canvas size is already known
(`:506-515`), and the size is recorded only while publishing is enabled
(`:553`).  So the first toggle of a session always returns `True`.  With a
missing device:

```
first toggle  -> True ("Video Out on"), last_error ''
after a frame -> not publishing, status "v4l2 unavailable", last_error '', panel OFF
second toggle -> False, last_error "/dev/video… unavailable (is the v4l2loopback module loaded?)"
```

The method's docstring describes exactly this bug as fixed.

**P2-15. The virtual camera is upside down.**  The frame tap delivers the
frame in GL row order, bottom row first.  Core's recorder and the streamer
both add `vflip` for that reason (`unicornviz/recording.py:826`).  The v4l2
sink writes the bytes unchanged (VERIFIED: bytes out equal bytes in), and
V4L2 consumers read the first row as the top.  The PipeWire path does flip
in its blit.  By reading; no loopback device was available to look at.

## P3

### midi-controllers-01 (sub-agent; VERIFIED unless noted)

| ID | Where | Finding |
|---|---|---|
| P3-1 | `apc_leds.py:672` | Bytes delivered together with a timeout return code are discarded.  libusb documents that this can happen; the effect would be an occasional lost pad press.  SUSPECTED on hardware |
| P3-2 | `apc_leds.py:1338`, `:1094`, `:1351` | With no APC, startup blocks 101 ms, makes 129 failed sends and logs two warnings; a 20 Hz state poll then runs for every user |
| P3-3 | `midi_manager.py:1440` vs `:581` | The panel says "Ctrl+Alt+M to close", but the modal absorbs that chord and stays open |
| P3-4 | `midi_manager.py:461-463`, `:581` | With no GL context or PIL the modal opens invisible and still absorbs keys.  By reading |
| P3-5 | `controller_profiles.py:190-204` | A NUL or DEL in a profile name is written unescaped and the saved file no longer parses |
| P3-6 | `controller_profiles.py` `parse_profile` | `"5"` and `"05"` collapse to one key; unknown action names and a negative `schema_version` are accepted |
| P3-7 | `apc_leds.py:366-406` | The raw-MIDI reader polls at 1 kHz forever, including after unplug; a message split across two reads is dropped and running status is not handled.  By reading |
| P3-8 | `apc_leds.py:84`, `:126`, `:757` | `global` statements (project rule: no global state) |
| P3-9 | `midi_manager.py:1325`, `:1340` | Private access across modules under `# noqa: SLF001` |
| P3-10 | `midi_manager.py:4-17`, `docs/operations.md:117-130` | Keys documented outside `HELP_ENTRIES`; the keys used inside the modals are in no `HELP_ENTRIES` at all |
| P3-11 | `apc_leds.py:486-487` | Permission denied on open is logged as "not found", at DEBUG |
| P3-12 | `controller_presets.py:328-399` | The help modal's pad text is hardcoded to the stock layout and still says "EQ/audio toggle" |
| P3-13 | `midi_manager.py:457-458`, `:1418` | The Control Room panel can read the modal's selection while it is being rebuilt.  SUSPECTED |
| P3-14 | `midi_manager.py:1582-1583`, `:459` | Modal state is discarded on close, which makes the selection-preserving code unreachable |
| P3-15 | `tests/` | Nine tests in two files; the USB driver, Learn, profiles and persistence are untested |

### video-out-01

| ID | Where | Finding |
|---|---|---|
| P3-16 | `video_out_controller.py:246-261` | `close()` runs on the calling thread and waits: 1.0 s when the queue was full at that moment (30 of 30 forced trials), 2.0 s when the writer is blocked in a write — VERIFIED.  It is reached from toggle-off and from a canvas-size change on the render thread |
| P3-17 | `:359-363`, `:321` | With the drop-in loaded and no backend enabled (the code default for both), a warning is logged at every launch |
| P3-18 | `:6-14` | The module docstring lists PipeWire as "planned"; it is implemented |
| P3-19 | `:436-444` | With no device configured, the first existing `/dev/video10`–`19` is used whether or not it is a loopback node |
| P3-20 | `:226-228` | A short write to the device is not detected, which would tear a frame.  By reading |
| P3-21 | `pipewire_sink.py:410-445` | Each frame creates and deletes a texture, and the call returns with framebuffer 0 bound and the texture binding changed, behind moderngl's back.  By reading; unreachable today (P2-13) |
| P3-22 | `install.sh:72-91` | Clones libfunnel from GitHub and builds it locally, pinned by a 7-character commit prefix |
| P3-23 | drop-in root | No `docs/` folder (the documentation SOP asks for four documents for a drop-in of this kind) |
| P3-24 | `:132-133`, `:211`, `:237` | The frame counters are updated from two threads without a lock; display-only |

## Recommended tests

Each is written to fail against the current code where a finding is open.
All use fake hardware; none needs a controller, a camera device or PipeWire.

### midi-controllers-01

| Test | Covers |
|---|---|
| With a fake libusb returning "no device", the reader makes a bounded number of calls per second and then stops or enters a reconnect state | P1-1 |
| After "no device" errors, the driver attempts a re-open and re-claim, and LED state is resent when it succeeds | P1-2 |
| `close()` does not call release or close until the in-flight transfer has returned | P2-1 |
| A learn message delivered on a non-main thread changes no map until the main thread drains | P2-2 |
| The note-on that completes a learn does not dispatch the newly bound action | P2-3 |
| Learn ignores events from a source other than the device being taught, and disarms after a timeout | P2-4 |
| Unbinding a pad, and switching to a profile that does not bind it, sends an off for that pad | P2-5 |
| A send that returns failure is retried once the port is available | P2-6 |
| With `device = ""`, no USB claim is made | P2-7 |
| The card matcher rejects `Launchkey MK2` and `MiniLab mk2` | P2-8 |
| A non-UTF-8 profile file is skipped and the others still load | P2-9 |
| Save then apply returns exactly the saved map, including cleared pads; a saved profile keeps both ports and its device | P2-10 |
| Two names that reduce to the same file are refused or made distinct; saving over a bundled key asks first | P2-11 |
| An open modal with unchanged state does not rebuild its image | P2-12 |
| No log record at INFO or above contains a note number | P2-2 |

### video-out-01

| Test | Covers |
|---|---|
| Loaded through `load_dropin_symbol` (no path changes), `submit_texture()` gets past the import | P2-13 |
| The first `toggle()` of a session with a missing device returns `False` and sets `last_error` | P2-14 |
| A frame whose first row is red and last row is blue reaches the device with the red row first in screen order | P2-15 |
| `close()` returns within a short bound when the queue is full and when the writer is blocked | P3-16 |
| Constructing the controller with no backend configured logs nothing above INFO | P3-17 |

## What works

- **MIDI is optional.**  The drop-in never imports `rtmidi` itself and every
  loader is guarded; the app starts without it.
- **LED traffic is small.**  Pads are diffed against what was last sent: 0
  messages over 20 idle ticks.  On the USB path writes happen on a writer
  thread, not the render thread.
- **Profile files.**  Names cannot escape the profiles folder, saves are
  temp-file-then-replace, a malformed TOML file is skipped with a warning,
  and the bundled stock profile matches the built-in preset.
- **Packet handling.**  USB-MIDI packing of SysEx and channel messages and
  the pad, scene, track and fader index helpers are correct by reading.
- **video-out v4l2 design.**  Frames go through a bounded queue to a writer
  thread and are dropped, with a counter, when the consumer is slow; a
  missing module or device produces one clear log line naming the setup
  script; the format structure is laid out correctly.
- **video-out off by default.**  Publishing does not start until asked.

## What this audit did not do

- Touch either controller, open a MIDI port, or claim a USB device.  Real
  unplug behavior, kernel-driver re-attach after a crash, LED colors and
  cable numbering need the hardware.
- Run either video-out backend against a real consumer (OBS, a loopback
  node, PipeWire).  P2-15 is from reading for that reason.
- Run `install.sh` or `v4l2loopback_setup.sh` (network, `sudo`).
- Launch the app.
- Fix anything.  The bandit findings are reported only.
