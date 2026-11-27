# Audio Out, CTA, OSC Bridge and Spotify audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

These are the last four drop-ins of the 2026-11-25/27 audit series.  With
this report every drop-in under `drop-ins/`, the core package and `tools/`
have been audited.

Scope:

- `drop-ins/audio-out-01` at `a07d3ae` (`0.6.0`): the output engine, its
  DSP and config validator (883 lines).
- `drop-ins/cta-01` at `21bc910` (`0.10.0`): the call-to-action controller,
  message editor and text field (1,497 lines).
- `drop-ins/osc-bridge-01` at `fd5fbe1` (`0.3.0`): the OSC control bridge
  and its validator (416 lines).
- `drop-ins/spotify-01` at `e14b6bb` (`1.0.0-rc.12`): the Spotify now-playing
  and Web API controller (1,082 lines).

Method:

- Every line was read.  No sub-agent was used.
- **audio-out** was run with a stand-in output stream, so no audio device
  was opened and nothing was played; the real callback and DSP were timed
  block by block.
- **OSC** was exercised by direct calls and with one real UDP server on a
  loopback-only ephemeral port.
- **Spotify** made no request to Spotify.  The Web API call was replaced
  with stand-ins, and the auth callback server was run on loopback ports.
  Endpoint shapes were **not** checked against Spotify's OpenAPI
  specification, which the project rules require for any endpoint claim, so
  this report makes none.
- **CTA** was loaded through core's own drop-in loader and its editor was
  drawn in an offscreen GL context.
- The app was not launched.  Timings are on this machine (**Mesa Intel Iris
  Xe**, 1920×1080).
- **VERIFIED** = reproduced by script.  **By reading** = found in the code,
  not run.  **SUSPECTED** = plausible, unconfirmed.  Scripts are
  `/var/tmp/uv-misc-audit/t14.py` and `t15.py`.

Severity: **P1** = crash, data loss or runaway.  **P2** = a feature that
does not work, wrong behavior the operator will notice, or a routine stall.
**P3** = minor or hygiene.

## Baseline

| Check | audio-out-01 | cta-01 | osc-bridge-01 | spotify-01 |
|---|---|---|---|---|
| Own tests | 24 passed | 29 passed | 15 passed | 133 passed |
| ruff | clean | clean | clean | clean |
| bandit (source) | 1 Low | 5 Low | clean | 6 Low |

Bandit, reported and not remediated (all Low):

| ID | Where | Assessment |
|---|---|---|
| B110 try/except/pass ×7 | audio-out ×1, cta ×3, spotify ×3 | Deliberate teardown and drawing fallbacks |
| B311 `random` ×2 | `cta_controller.py`, `cta_editor.py` | False positives: message order and sparkle placement |
| B105 "hardcoded password" | `spotify_controller.py` | False positive on the token URL constant |
| B404, B603 | `spotify_controller.py` | `playerctl` run by resolved path with an argument list, no shell.  Acceptable |

`spotify_controller.py` carries three `# nosec B310` annotations (`:837`,
`:863`, `:920`) on requests to fixed `https://` Spotify URLs.  The owner
approved the equivalent annotation in lyrics-01 on 2026-11-27; these three
are listed for the same decision.

## Summary

No P1.

| ID | Sev | Drop-in | Finding |
|---|---|---|---|
| P2-1 | P2 | spotify | Without `playerctl` (Windows), the Web API result never reaches the now-playing fields — VERIFIED |
| P2-2 | P2 | spotify | A revoked token fails every poll forever and never asks for a new login — VERIFIED |
| P2-3 | P2 | spotify | If the callback port is busy, login dies silently and reports "auth started" — VERIFIED |
| P2-4 | P2 | cta | The message editor costs 90 ms per frame with its default effects — VERIFIED |
| P2-5 | P2 | cta | Edited messages are not saved; they are gone at the next launch — VERIFIED |
| P2-6 | P2 | audio-out | Turning the engine off removes its Control Room panel for good — VERIFIED |
| P2-7 | P2 | audio-out | Starting the engine loads and resamples every sample on the calling thread: 2 s — VERIFIED |
| P2-8 | P2 | osc | Inbound values are neither bounded nor rate-limited, and there is no sender check — VERIFIED |

Plus 22 P3 items.

## P2

### P2-1. Spotify Web API results never reach the display — VERIFIED

`_refresh_remote_state()` stores the `/me/player` payload in `_playback`
(`spotify_controller.py:967-1019`) and nothing copies it into `_title`,
`_artist`, `_track_id`, `_is_playing` or `_available`.  Those fields are
only ever written by the `playerctl` path (`:707-750`).  The docstring at
`:379-380` says that without `playerctl` "the web API is the primary track
detector".

With local polling off and one successful stand-in poll:

```
title '' artist '' is_playing False available False | playback item held: Song Title
```

So on Windows, where the drop-in itself logs that `playerctl` is not
supported and tells the user to enable the Web API (`:155-159`), the HUD,
the banner and the lyrics overlay get nothing.

### P2-2. A revoked token never asks for a new login — VERIFIED

On HTTP 401 `_api_get()` sets `auth_status = 'token_expired'` and returns
(`:927-928`).  `_auth_ready` stays true, a refresh is attempted only when
the stored expiry time has passed (`:1030-1032`), and `token_expired` is not
in the set of states that raise the "needs auth" flag (`:495-505`).

Three polls with a token the server rejects: `auth_status token_expired |
auth_ready True | needs_auth False`.  The controller keeps polling and
failing until the clock expiry, up to an hour, and the operator is never
told to log in again.

### P2-3. Login dies silently when the callback port is busy — VERIFIED

`begin_auth()` creates the loopback HTTP server before its `try` block
(`:598`).  If port 43879 is already taken, the `OSError` escapes the worker
thread.  `begin_auth_async()` has already returned "Spotify auth started",
and the status stays `awaiting_user_auth`.

Reproduced on a busy loopback port: `message 'Spotify auth started' | worker
thread died with ['OSError'] | auth_status left at awaiting_user_auth`.

Related: the server accepts the first GET on **any** path as the callback
(`:48-56`).  A request for `/favicon.ico` arriving first ended the login
with `auth_path_mismatch` — VERIFIED.  The state check does protect the
token exchange itself.

### P2-4. The CTA editor costs 90 ms per frame — VERIFIED

With the default effects on, `_update_fx()` marks the editor dirty every
frame (`cta_editor.py:448-449`), and each dirty frame redraws a full-window
image (gradients built per box, text measured per character), converts it
and uploads it (`:409-425`, `:659-845`).

| Editor state | Median | Max |
|---|---|---|
| Effects on (default) | **89.8 ms** | 185.3 ms |
| Effects off | 1.4 ms | 95.6 ms (on a redraw) |

With the editor open the whole app runs at about 11 fps, on the audience
output.  The config editor does offer the switch, and its hint says so
("re-rasters the editor every frame"); the default is the expensive one.

### P2-5. Edited CTA messages are not saved — VERIFIED

`_apply_editor_result()` replaces the in-memory list
(`cta_controller.py:305-313`) and returns "CTA: messages saved".  The only
runtime-state write in the drop-in is `cta.editor_open`.  After an edit and
Ctrl+S, a fresh controller still had the original message.  The list comes
back from `[cta] messages` in config, or the three built-in defaults, at
every launch.

### P2-6. Audio-out loses its Control Room panel on toggle-off — VERIFIED

`shutdown()` unregisters the panel (`audio_out_controller.py:587-592`), and
`toggle_enabled()` calls `shutdown()` to stop the stream (`:489-492`).  The
panel is registered once, in the constructor.  After one off/on cycle the
unregister had run and nothing had re-registered it, so the panel that
holds the TOGGLE button is gone until restart.  The hotkey still works.

### P2-7. Starting audio-out stalls for seconds — VERIFIED

`_start()` reads every SFX file and the music bed, resamples each to the
stream rate, and only then opens the stream (`:263-266`, `:301-336`), all on
the thread that pressed the key.  With twelve two-second SFX and a
four-minute music bed recorded at 48 kHz: **1,962 ms**, and 2,043 ms again
on the next toggle-on, because nothing is kept between starts.  The music
bed is held decoded in memory (85 MB for four minutes).

The audio callback itself is within budget: 0.7 ms per 1,024-frame block
with filters open, 1.2 ms with filters and EQ engaged, 6.5 ms (max 9.1) with
reverb and eight voices, against 23.2 ms available.  Almost all of the
reverb cost is the four comb filters.

### P2-8. OSC input is unbounded and unchecked — VERIFIED

- **Values pass straight through.**  `/uv/reactivity` with `NaN` and with
  `1e30` both reached `vj_api.set_reactivity()` unchanged
  (`osc_bridge_controller.py:202-204`); `/uv/postfx/slot -3` reached
  `select_postfx_slot(-3)`.  Only the beat-flash brightness is clamped, by
  the governor.
- **No queue limit, no per-frame limit.**  Every message is queued and the
  whole queue is drained each frame (`:278-288`).  200,000 queued messages
  took 104 ms in one frame.
- **No sender check.**  OSC is plain UDP.  The default bind is
  `127.0.0.1`, which limits this to local programs, but driving the show
  from TouchOSC on a tablet, the documented use, needs a LAN address; the
  validator accepts `0.0.0.0` without comment.  Anyone on that network can
  then change effects, fire SFX and set parameters.
- The last address received is kept at any length (60,001 characters
  tested) for the Control Room row.

## P3

### spotify-01

| ID | Where | Finding |
|---|---|---|
| P3-1 | `spotify_controller.py:84`, `:676`, `:1021` | The class says it is updated on the main thread only; two worker threads write auth status, tokens and errors with no lock |
| P3-2 | `:647-674`, `:873-898` | A token refresh that completes just after logout writes the token file back and marks the session ready again — VERIFIED |
| P3-3 | `:799-808` | The token file is written in place and only then set to mode 600; an interrupted write leaves a truncated file, reported as `token_error` |
| P3-4 | `:773-778` | The token path is resolved from the source file's location, not the app root; an absolute `token_store` may point anywhere |
| P3-5 | `:147-150` | `now_playing_banner_delay_s = 0` becomes 15.0 — VERIFIED |
| P3-6 | `:263-268` | Four scopes are requested by default.  Whether all four are needed was not checked against the specification |
| P3-7 | `:603` | Login opens the system browser from the show machine with no warning on the audience output |

### cta-01

| ID | Where | Finding |
|---|---|---|
| P3-8 | `cta_editor.py:633-655` | The editor uses a list of system font paths instead of the bundled UI font; the default icons (thumbs up, bell, outbox) are missing-glyph boxes in the font found here — VERIFIED |
| P3-9 | `text_field.py:132-133` | A field already longer than its limit accepts part of a multi-character input: 125 characters plus a 10-character event became 130 — VERIFIED.  Config messages over 120 characters trigger it |
| P3-10 | `cta_controller.py:117` | `mode = "random"` with the default seed gives the same order every session (`d, d, a, c, e` twice) — VERIFIED |
| P3-11 | `cta_controller.py:339-340`, `:206-207` | "CTA: cooldown" is shown when CTA is disabled; the panel says "on cooldown or disabled" |
| P3-12 | `cta_editor.py:270-272` | Escape discards all edits with no confirm |
| P3-13 | `cta_editor.py:699-715`, `:784-834` | Key hints are drawn inside the editor, outside `HELP_ENTRIES`; the editor's own keys are in no `HELP_ENTRIES` |

### audio-out-01

| ID | Where | Finding |
|---|---|---|
| P3-14 | `audio_out_controller.py:517-523` | With `beat_trigger` on, each SFX is played into the output the app is listening to, so an SFX can be detected as a beat and trigger the next one.  SUSPECTED; depends on the capture source |
| P3-15 | `:476-477` | An unknown SFX name plays a random sample instead of nothing — VERIFIED |
| P3-16 | `:243-251`, `:323-328` | `sfx_dir` and `music_file` are resolved from the source file's location and may point anywhere |
| P3-17 | `:362-364` | A failing callback logs an error on every block (about 43 a second) |
| P3-18 | `config_validator.py:41-43` | Booleans pass as numbers; `samplerate`, `blocksize` and `max_voices` have no range check |

### osc-bridge-01

| ID | Where | Finding |
|---|---|---|
| P3-19 | `osc_bridge_controller.py:339-343`, `:372-374` | Toggle-off waits for the server loop on the calling thread: 300–400 ms — VERIFIED |
| P3-20 | `:198`, `:223-226` | The address is lower-cased before the parameter name is taken from it, so `/uv/effect/param/GlowAmount` sets `glowamount` — VERIFIED |
| P3-21 | `:271-275` | Reaches into the beat-flash controller's private `_governor`; `shutdown()` does not unregister the Control Room panel |
| P3-22 | `config_validator.py:24-30` | `port = true` passes validation — VERIFIED |

## Spotify rules check

Against the project's Spotify Web API rules, by reading:

| Rule | Status |
|---|---|
| Authorization Code with PKCE, no Implicit Grant | Met (`:628-645`, S256 challenge) |
| Loopback redirect on `http://127.0.0.1` | Met (default `http://127.0.0.1:43879/callback`) |
| No client secret in runtime code | Met; client ID only |
| Tokens in an ignored local runtime file, with refresh | Met (`runtime/spotify-token.json`, mode 600, refresh at `:845-871`); see P2-2 for the revoked-token gap |
| Handle 429, honor `Retry-After`, back off | Met (`:929-945`) |
| Surface meaningful operator feedback on errors | Partly: see P2-2 and P2-3 |
| No caching of Spotify content beyond runtime | Met; only the token is persisted |
| Endpoint paths and fields per the OpenAPI spec | **Not checked** (no network, spec not consulted) |

## Recommended tests

Each is written to fail against the current code where a finding is open.
None needs a network, an audio device or a Spotify account.

### spotify-01

| Test | Covers |
|---|---|
| With local polling off, a stand-in `/me/player` payload sets title, artist, playing state and `available` in `snapshot()` | P2-1 |
| After a 401, the next poll attempts a refresh, and a failed refresh raises `needs_auth` | P2-2 |
| `begin_auth_async()` with the callback port in use ends in a failure status, with no uncaught exception | P2-3 |
| A request to any path other than the callback path does not end the login | P2-3 |
| A refresh completing after `logout()` leaves no token file | P3-2 |
| An interrupted token write leaves the previous token readable | P3-3 |

### cta-01

| Test | Covers |
|---|---|
| With the editor open and no input, the image is not rebuilt every frame (or stays under a stated time at 1080p) | P2-4 |
| Messages saved in the editor are returned by a newly constructed controller | P2-5 |
| A field over its limit accepts no further input | P3-9 |
| Two controllers in random mode with no seed do not produce the same sequence | P3-10 |
| A disabled controller reports "disabled", not "cooldown" | P3-11 |

### audio-out-01

| Test | Covers |
|---|---|
| After toggle off then on, the Control Room panel is registered | P2-6 |
| A second start does not re-read or resample files already loaded; a start returns within a bound with loading deferred | P2-7 |
| `trigger_sfx('unknown')` plays nothing and returns `None` | P3-15 |
| The callback stays under half its block period with reverb and the maximum voice count | holds today; unpinned |
| Validator rejects booleans and out-of-range `blocksize` and `max_voices` | P3-18 |

### osc-bridge-01

| Test | Covers |
|---|---|
| `NaN`, infinity and out-of-range values are clamped or dropped before reaching `vj_api` | P2-8 |
| With 10,000 messages queued, one frame runs a bounded number and the queue never exceeds a cap | P2-8 |
| A non-loopback `host` requires an explicit allow setting (or an allowed-sender list) | P2-8 |
| `/uv/effect/param/GlowAmount` sets `GlowAmount` | P3-20 |
| Toggle-off returns within a short bound | P3-19 |

## What works

- **Spotify auth design.**  PKCE with a fresh verifier and state per
  attempt, a loopback redirect, no secret, a state check before the token
  exchange, and rate-limit handling that honors `Retry-After`.
- **Spotify polling.**  Both the `playerctl` poll and the Web API poll run
  on worker threads, and a track change triggers the Web API poll instead of
  a fixed fast cadence.
- **Audio callback.**  Well inside its budget in every configuration tried,
  with filter state carried across blocks and parameters smoothed.
- **OSC threading.**  The server thread only queues; every action runs on
  the main thread, and a failing action is logged without stopping the
  rest.  All the `vj_api` names it calls exist.
- **CTA editor loading.**  The relative import works through core's loader,
  and the text fields use real text-input events.
- **Independence.**  All four degrade cleanly when their optional library
  (`sounddevice`, `python-osc`, `playerctl`) is missing.

## What this audit did not do

- Contact Spotify, or check any endpoint against its specification.
- Open an audio device or play sound.
- Accept OSC from another machine.
- Launch the app or type into the CTA editor through real SDL input.
- Test Windows.
- Fix anything.  The bandit findings and the `# nosec` annotations are
  reported only.
