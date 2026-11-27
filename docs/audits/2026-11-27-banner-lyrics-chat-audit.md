# Banner, Lyrics and Chat audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/banner-01` at `7215723` (`0.14.0`): the bottom banner and its
  config modal and text editor (1,367 lines).
- `drop-ins/lyrics-01` at `c9a1747` (`0.5.0`): the synced-lyrics overlay,
  LRC helpers and config validator (678 lines).
- `drop-ins/chat-01` at `62fc55d` (`0.7.1`): the chat overlay controller
  (787 lines) and the web widget (234 lines of TSX).
- Core's side of all three: key dispatch, overlay dispatch, runtime state.

Method:

- Every line was read, including the web widget.  No sub-agent was used.
- Each controller was run in an offscreen GL context with a stand-in for
  `vj_api`: the chat renderer with and without initialization, the lyrics
  clock through play, pause and resume at three polling setups, and the
  banner over 600 frames in each mode, with the modal open, and through
  editing, pasting and damaged saved state.
- No network request was made.  The `ably` package is not installed in this
  environment, so the Ably connection itself was not exercised; the message
  handler was called directly.  `playerctl` was not run.
- The app was not launched.  Timings are on **Mesa Intel Iris Xe** at
  1920×1080.
- **VERIFIED** = reproduced by script.  **By reading** = found in the code,
  not run.  Scripts are in `/var/tmp/uv-misc-audit/` (`t4.py`, `t5.py`).

Severity: **P1** = crash, data loss or runaway.  **P2** = a feature that
does not work, wrong behavior the audience or operator will notice, or a
routine frame-time hit.  **P3** = minor or hygiene.

## Baseline

| Check | banner-01 | lyrics-01 | chat-01 |
|---|---|---|---|
| Own tests | 1 passed | 17 passed | 10 passed |
| ruff | clean | clean | clean |
| bandit (source) | 3 Low | 7 Low | 2 Low |

Bandit, reported and not remediated (all Low):

| ID | Where | Assessment |
|---|---|---|
| B110 try/except/pass ×6 | `banner_controller.py:503`, `:1280`, `:1356`; `chat_controller.py:380`, `:780`; `lyrics_controller.py:549` | Deliberate: clipboard, teardown and panel unregister |
| B112 try/except/continue | `lyrics_controller.py:454` | Deliberate: font fallback loop |
| B404, B603 ×2, B607 ×2 | `lyrics_controller.py:29`, `:262`, `:271` | `playerctl` is run by bare name with an argument list and no shell.  The player name comes from config.  Acceptable; a full path would close B607 |

`lyrics_controller.py:336` carries a `# nosec B310` annotation on the LRCLIB
request.  Project rules reserve `# nosec` for owner-approved cases; it is
listed here for that decision (the code does check for `https://` first).

## Summary

No P1.

| ID | Sev | Drop-in | Finding |
|---|---|---|---|
| P2-1 | P2 | chat | The chat overlay cannot draw anything: three separate faults — VERIFIED |
| P2-2 | P2 | chat | With chat enabled, plain `C` no longer opens the config editor — by reading |
| P2-3 | P2 | chat | No sender identity, rate limit or moderation, and the root API key is meant to live in the committed `config.toml` — partly VERIFIED |
| P2-4 | P2 | lyrics | Lyrics lose sync on pause and resume at fast poll settings or with a coarse position source — VERIFIED |
| P2-5 | P2 | lyrics | Long lines are cut off at the screen edge; `max_lines` does nothing — VERIFIED |
| P2-6 | P2 | banner | The banner re-rasters on half of all frames at about 11 ms each; the config modal costs 26 ms per frame — VERIFIED |
| P2-7 | P2 | banner | Lines wider than the window are cut off — VERIFIED |
| P2-8 | P2 | banner | Lowering the max line length rewraps the text permanently — VERIFIED |

Plus 22 P3 items.

## P2

### P2-1. The chat overlay cannot draw — VERIFIED

Three independent faults, any one of which is enough:

1. **The renderer is never created.**  `ChatController.init_gl()`
   (`chat_controller.py:524-534`) is the only place the renderer is built,
   and nothing in core or the drop-in calls it.  After construction the
   renderer is `None` and `render_overlay()` returns at `:565`.  With a
   message queued, 0 pixels changed.
2. **The first draw would raise.**  `render_panel()` and
   `render_text_line()` use `moderngl.SCISSOR_TEST` (`:295`, `:364`), which
   does not exist in moderngl.  Calling `init_gl()` by hand and rendering:
   `AttributeError: module 'moderngl' has no attribute 'SCISSOR_TEST'`.
3. **The text pass does not draw text.**  The glyph buffer holds four
   floats per vertex (position plus atlas cell, `:352-357`), but the vertex
   array is declared with a two-float position only (`:290`), so every other
   "vertex" is an atlas index read as a screen position.  The fragment
   shader (`:203-230`) has no sampler and never reads the font texture.
   With the missing constant stubbed, one bottom-band message changed
   **874,000 pixels outside the band** (46% of the screen) with solid color.

The background panel also depends on blending being on, which the renderer
does not set: with it off the panel is opaque black.

Chat is off in the shipped config, which is why this has not been seen.  The
drop-in's 10 tests cover message intake and the panel, not drawing.

### P2-2. With chat enabled, `C` stops opening the config editor — by reading

Drop-in key handlers run before core's own keys, and the first one that
returns anything but `False` ends dispatch (`hotkeys.py:990-1000`).  Chat
claims plain `C`, Ctrl+C and the keypad digits (`chat_controller.py:734-765`).
Core's "naked C — toggle the configuration editor" is further down
(`hotkeys.py:1839-1843`) and is never reached.  The help overlay then lists
`C` twice with two meanings.

### P2-3. Chat has no identity, rate limit or moderation — partly VERIFIED

Inbound text is safely reduced to printable ASCII and capped at 32 and 280
characters; that part is tested and holds.  What is missing:

- **Names are whatever the sender says.**  A message carrying
  `username: "VJ"` is queued as "VJ" — VERIFIED.  Nothing marks the
  operator's own name as reserved.
- **No rate limit and no filter.**  The band shows the last 8 messages and
  the buffer holds 40, so one sender can replace everything on screen.
  There is no word filter, block list or operator "clear" control.
- **The key.**  The Python side reads the Ably **root** key from
  `[chat] ably_api_key` (`:437`), and `docs/backend-requirements.md` says it
  is "stored in `config.toml`".  `config.toml` is a tracked file in this
  repository, so a key placed there is one commit away from being published.
  The web widget accepts the same root key as a prop for "dev/staging"
  (`ChatWidget.tsx:32-33`), and the token endpoint that replaces it is
  marked "pending implementation".

None of this is live while P2-1 stands, but it is what goes on the venue
screen the moment the overlay works.

### P2-4. Lyrics lose sync on pause and resume — VERIFIED

The position estimate re-anchors only when two consecutive polls differ by
more than 0.3 s (`lyrics_controller.py:393-396`).  While paused it shows the
last anchor (`:353-356`), and the wall-clock reference is not reset when
playback resumes.

Clock error in seconds (shown position minus true position):

| Setup | After 30 s play | Paused 10 s | 0.5 s after resume | 5 s after resume |
|---|---|---|---|---|
| Poll 0.5 s (default), exact position | 0.0 | 0.0 | 0.0 | 0.0 |
| Poll 0.2 s (lowest setting), exact position | 0.0 | **−30.0** | **+10.0** | **+10.0** |
| Poll 0.5 s, source position updates every 5 s | −0.3 | **−5.0** | −0.3 | −0.5 |

At a poll interval of 0.3 s or less the estimate never re-anchors, so a
pause jumps the lyric back to the start of the track and a resume leaves it
ahead by the length of the pause, permanently.  The config editor offers
0.2 s as the low end of the range.  With a position source that updates in
steps, pausing shows a line up to one step old.

### P2-5. Long lyric lines are cut off — VERIFIED

`_refresh_texture()` (`:458-482`) clamps the texture to the canvas width and
draws the text from the left padding; there is no wrap and no shrink.  A
137-character line at the default 40 px needs 3,336 px and gets 1,280 on a
1,280-wide canvas, so most of it is missing.  `max_lines` is read and
clamped (`:120`) and never used: one line is always drawn.

### P2-6. Banner frame cost — VERIFIED

The banner is a full-width image drawn with PIL, converted and uploaded
whenever its cache key changes (`banner_controller.py:1312-1337`).  The key
includes the fade step (48 steps a second) and the text color, which moves
continuously with beat color on.

600 frames at 60 fps, 1920 wide:

| Mode | Frames re-rastered | Median | p95 | Max |
|---|---|---|---|---|
| Beat color on (default) | 307 of 600 | 3.7 ms | 11.7 ms | 16.9 ms |
| Beat color off | 218 of 600 | 0.8 ms | 11.3 ms | 17.9 ms |
| Drip on | 600 of 600 | 8.8 ms | 19.1 ms | 34.7 ms |
| Config modal open | every frame | 26.5 ms | — | 68.3 ms |

So with the banner on, about half of all frames pay roughly 11 ms of a
16.7 ms budget on the render thread, and opening the modal takes the app
below 40 fps by itself.  The modal asks for three fonts and redraws the whole
panel each frame (`:962-1040`, `:1341-1345`).

### P2-7. Banner lines wider than the window are cut off — VERIFIED

A line is centered, or pinned to the left padding when it is too wide
(`:954-955`); nothing scales or wraps to the pixel width.  `max_line_chars`
is a character count (20 to 200) and the font goes up to 56 px.

- Default text at the default 28 px: lines are 1,662, 1,528, 1,629 and
  587 px wide.  On a 1,280-wide window **3 of 4 lines** lose their end.
- At 56 px the same lines are about 3,300 px: 3 of 4 are cut off at 1,920.

### P2-8. Changing the max line length rewraps the text for good — VERIFIED

`_set_max_line_chars()` rewraps any over-long line and stores the result
(`:623-630`); nothing rejoins lines when the limit goes back up.  In the
config modal the Left and Right arrow keys change the limit by 10.

Default text, limit taken from 100 to 20 and back to 100: **4 lines → 29
lines → 29 lines**, text not restored.  The operator's own line breaks are
lost the same way, and every step is saved to the runtime state.

## P3

### banner-01

| ID | Where | Finding |
|---|---|---|
| P3-1 | `banner_controller.py:525-529`, `:686-719` | Every keystroke in the editor saves the whole runtime state file to disk on the render thread (11 writes for 11 characters — VERIFIED; `runtime_state.py:124-133` persists immediately) |
| P3-2 | `:351-387` | The per-line cap limits each line but not the number of lines.  Pasting 50,000 short lines is accepted (539,000 characters), saved to runtime state, and makes each frame 75 ms with the modal open — VERIFIED |
| P3-3 | `:533-553` | A non-numeric `max_line_chars`, `font_px` or `alpha` in the saved state raises in the constructor, so the banner fails to load — VERIFIED |
| P3-4 | `:1296-1310` | `render_overlay()` reads `app._primary_display_viewport` (private) and forces the draw onto the screen framebuffer regardless of what core had bound |
| P3-5 | `:47-59` vs `:777-780` | Alt+B (beat color) works and is shown in the modal, but is missing from `HELP_ENTRIES` |
| P3-6 | `:312-318`, `:721-727`, `:785-791` | Reset replaces the operator's text with the default in one press, with no confirm and no undo; it is also a Control Room button |
| P3-7 | `:321-323` | The MIDI toggle does not save the new on/off state; the hotkey and panel do |
| P3-8 | `:898-899`, tests | Leaves blending enabled after drawing.  One test for 1,367 lines; the text editor is untested |

### lyrics-01

| ID | Where | Finding |
|---|---|---|
| P3-9 | `lyrics_controller.py:223-238` | Lyrics only follow the Spotify subsystem or `playerctl`.  The app's own media player and mixer publish to core's now-playing hub and are not consulted, so local tracks never get lyrics |
| P3-10 | `:58-63`, `:449-456` | Hiragana and hangul render as the missing-glyph box in the bundled font — VERIFIED.  Cyrillic is fine |
| P3-11 | `:56` | The User-Agent sent to lrclib.net names `github.com/iDoMeteor/unicorn-viz`, which is not this repository's remote |
| P3-12 | `:328-346` | A network failure is recorded as "no lyrics" for that track, so nothing is retried until the track changes |
| P3-13 | `lrc.py:13-34` | Lookup is exact-match only (artist, title, album, duration); there is no fallback to LRCLIB's search for titles such as "… - Remastered 2011" |
| P3-14 | `:125-129` | The default cache folder is found by walking up from the source file, not from the app root |
| P3-15 | `config_validator.py:16-33` | `poll_interval_s` is not validated; booleans pass as numbers.  The controller ignores `enabled`, which the validator checks |
| P3-16 | `:419`, `:503` | Falls back to `app._ctx` (private); leaves blending enabled after drawing |
| P3-17 | `:240-283` | `playerctl` is Linux-only; on Windows the fallback silently returns nothing |

### chat-01

| ID | Where | Finding |
|---|---|---|
| P3-18 | `chat_controller.py:407-428`, README | The "Chat username" setting is described as "name your messages are posted as", but the app has no code that sends a message |
| P3-19 | `:445-446` | `glitter = 0` becomes 0.7 and `pulse = 0` becomes 0.5 — VERIFIED; neither can be turned off |
| P3-20 | `:172` | When the 256-message queue is full, the receive callback blocks the network thread — VERIFIED |
| P3-21 | `:233-242` | Non-ASCII names and text become spaces by design, so a visitor writing in another script shows as blanks |
| P3-22 | `web/ChatWidget.tsx:73-81`, `:100-106` | The widget adds its own message locally and also receives it back from the channel, so each sent message appears twice.  By reading |

### Shared

- **Network at runtime.**  Project rules say the main application makes no
  network requests at runtime.  Lyrics (lrclib.net, sending the artist and
  title being played) and chat (Ably) both do once switched on.  Both are
  off until the operator enables them.  Listed for the owner to confirm the
  rule's intent.
- **Bottom edge.**  The banner and the chat band's default position both
  occupy the bottom of the screen and would overlap.
- **Docs.**  Chat has `docs/backend-requirements.md` only; the documentation
  SOP asks for operations, configuration, integration and troubleshooting.
  Chat and lyrics both list hotkeys in docstrings, which the rules reserve
  for `HELP_ENTRIES`.

## Recommended tests

Each is written to fail against the current code where a finding is open.

### chat-01

| Test | Covers | Kind |
|---|---|---|
| Core's startup path leaves an enabled chat controller with a renderer | P2-1 | Python (fake context) |
| With one message queued, `render_overlay()` changes pixels only inside the panel rectangle, and the result differs for two different messages | P2-1 | GL |
| Every moderngl constant the module references exists | P2-1 | Python |
| With chat enabled, plain `C` still reaches the config editor (or the help overlay lists a single meaning for it) | P2-2 | Python |
| A message claiming the operator's configured username is rejected or marked | P2-3 | Python |
| More than N messages from one sender inside a second are dropped | P2-3 | Python |
| `glitter = 0` and `pulse = 0` are honored | P3-19 | Python |
| A full queue drops the message without blocking the callback | P3-20 | Python |

### lyrics-01

| Test | Covers | Kind |
|---|---|---|
| Play 30 s, pause 10 s, resume: the position estimate stays within half a second of the true position, at poll intervals of 0.2, 0.5 and 2 s | P2-4 | Python (fake clock) |
| The same with a source whose position advances in 5 s steps | P2-4 | Python (fake clock) |
| A line wider than the canvas is wrapped or scaled so its last word is drawn | P2-5 | Python (PIL) |
| `max_lines = 2` draws two lines | P2-5 | Python (PIL) |
| With no Spotify subsystem, the now-playing hub's track is used | P3-9 | Python |
| A failed fetch is retried after a delay for the same track | P3-12 | Python |
| The validator rejects a non-numeric or out-of-range `poll_interval_s` | P3-15 | Python |

### banner-01

| Test | Covers | Kind |
|---|---|---|
| Over 600 frames with a steady line on screen, the banner image is rebuilt only when the line or its fade step changes, and not for color alone | P2-6 | Python (fake context, counting) |
| With the modal open and nothing changing, the modal image is not rebuilt | P2-6 | Python (counting) |
| For window widths of 1,280 and 1,920 and font sizes 16, 28 and 56, the last character of every displayed line falls inside the image | P2-7 | Python (PIL) |
| Max line 100 → 20 → 100 returns the original text | P2-8 | Python |
| Typing ten characters writes runtime state at most once (on leaving edit mode or after a pause) | P3-1 | Python |
| A paste beyond a stated total size is truncated | P3-2 | Python |
| Saved state with non-numeric values falls back to defaults | P3-3 | Python |
| Text editor: cursor movement, selection, word jump, cut/copy/paste and the per-line cap, by direct calls to the `_tf_*` methods | untested today | Python |
| Every key the handler acts on appears in `HELP_ENTRIES` | P3-5 | Python |

## What works

- **Chat intake.**  Inbound text is truncated, coerced and limited to
  printable ASCII before it can reach a draw call; malformed payloads are
  ignored.
- **Lyrics at the default settings.**  With a 0.5 s poll and an exact
  position the clock stays within 0.02 s through play, pause and resume.
  Fetching and `playerctl` both run off the render thread, results are
  cached on disk, and a line change costs 3–14 ms once.
- **LRC parsing.**  Multiple timestamps per line, metadata tags and blank
  lines are handled, and the active line is found by bisection.
- **Banner text editor.**  Uses real text-input events, so layouts, dead
  keys and non-ASCII input work, with selection, word jumps and the
  clipboard; over-long lines scroll to follow the cursor.
- **Banner when static.**  With beat color off and no fade in progress the
  cached texture is reused and a frame costs 0.8 ms.
- **Independence.**  All three load through guarded loaders and register
  their panels and MIDI actions defensively.

## What this audit did not do

- Connect to Ably or to lrclib.net, or run `playerctl`.
- Launch the app, open a window, or type into the banner editor through
  real SDL input.
- Build or run the web widget; it was read only.
- Test Windows.
- Fix anything.  The bandit findings and the `# nosec` annotation are
  reported only.
