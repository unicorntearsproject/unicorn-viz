# Media player and Image Showcase audit (2026-11-27)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-27

Scope:

- `drop-ins/media-01` at `2c85458` (`0.31.1`): the controller and playback
  backends (2,487 lines), the Media Deck window (2,614), the playlist store
  (319) and the tag reader/writer (164).
- `drop-ins/images-01` at `cf1a52e` (`0.9.0`): the Image Showcase effect
  (510 lines).
- Core's call sites for both (startup prefetch/prescan, `warm_cache`).

Method:

- Every line was read.  The controller and images-01 were read directly; one
  read-only sub-agent read the deck window, playlist store and tag module,
  and its three most serious findings were then reproduced independently.
- The controller was driven with a fake playback backend against scratch
  libraries of empty dummy files (30 and 10,800 tracks).  No audio was
  played and VLC was never started.
- The playlist store and tag module were exercised on scratch files; the
  deck window's logic was exercised with a fake GL window.
- Image Showcase was run in an offscreen GL context on generated images.
- Nothing under the live checkout, `runtime/`, or `~/Music` was read or
  written.  The app was not launched.  Timings are on this machine (16 CPU
  threads, **Mesa Intel Iris Xe**).
- **VERIFIED** = reproduced by script.  **By reading** = found in the code,
  not run.  **SUSPECTED** = plausible, unconfirmed.  Scripts are in
  `/var/tmp/uv-media-audit/` (`mine/` and `ui/`).

Severity: **P1** = data loss, or runaway behavior that needs a restart.
**P2** = wrong behavior the operator or audience will notice, or a routine
stall.  **P3** = minor or hygiene.

## Baseline

| Check | media-01 | images-01 |
|---|---|---|
| Own tests | 174 passed (125 s) | none in the drop-in |
| ruff | clean | clean |
| bandit (source) | 25 Low | clean |

Bandit, reported and not remediated (all Low, all in media-01):

| ID | Count | Where | Assessment |
|---|---|---|---|
| B110 try/except/pass | 20 | controller 16, `ui.py` 3, `tags.py` 1 | Mostly deliberate best-effort teardown and VLC calls; several hide failures the operator should see (see P3-9) |
| B112 try/except/continue | 2 | `media_controller.py:186`, `:301` | Deliberate fallbacks |
| B404 / B603 subprocess | 2 | `media_controller.py:29`, `:666` | Argument list, no shell; the path comes from the library scan.  Acceptable |
| B101 assert | 1 | `media_controller.py:1637` | Guards an import spec; harmless |
| B105 hardcoded password | 1 | `media_controller.py:2393` | False positive: the key is `token_expires_in_s` and the value is `0.0` |

Images-01 decodes arbitrary user images with Pillow, so the 42 open Pillow
advisories from the core audit apply to it directly.

## Summary

| ID | Sev | Where | Finding |
|---|---|---|---|
| P1-1 | P1 | playlist store | A damaged store file is replaced by an empty one at the next track change: every playlist, favorite and saved setting is lost — VERIFIED |
| P1-2 | P1 | controller | When tracks cannot be played, the player skips one per frame and starts a measurement thread for each — VERIFIED |
| P2-1 | P2 | controller + deck | Deleting a file or saving tags rescans and reorders the library; the "current track", the queue and the selection then point at different tracks, and the wrong file path is published — VERIFIED |
| P2-2 | P2 | controller | `boot_playlist` and the remembered last track are dropped whenever the startup scan is still running, which is the normal startup — VERIFIED |
| P2-3 | P2 | deck window | Drawing the deck walks the whole library several times per frame: 206 ms per frame at 10,000 tracks — VERIFIED |
| P2-4 | P2 | playlist store | Deleted and renamed playlists come back at the next launch — VERIFIED |
| P2-5 | P2 | tag editor | Saving the tag editor collapses multi-value tags to the first value — VERIFIED |
| P2-6 | P2 | controller | With crossfade on, tracks shorter than the crossfade are skipped on their first frame — VERIFIED |
| P2-7 | P2 | controller | Auto-level decodes the whole file into memory: about 59 MB per minute of audio — VERIFIED |
| P2-8 | P2 | deck window | Text entry cannot type shifted symbols, ignores Caps Lock and has no paste — VERIFIED |
| P2-9 | P2 | deck window | Four selection and input faults (search box swallows hotkeys; sort keeps the row, not the track; clicks pass through flyouts; sideways wheel scrolls down) — VERIFIED |
| P2-10 | P2 | deck window | "Prune missing" runs without a confirm and empties a playlist when the music drive is unmounted — by reading |
| P2-11 | P2 | controller | The last-resort player `aplay` plays compressed files as raw noise — by reading |
| P2-12 | P2 | images | Past 128 images, whole folders are never shown, and activations that pick them show a mix of other folders — VERIFIED |
| P2-13 | P2 | images | Photo orientation (EXIF) is ignored: phone photos show sideways — VERIFIED |
| P2-14 | P2 | images | Without preload, first activation decodes on the render thread: 287 ms per 12-megapixel photo — VERIFIED |
| P2-15 | P2 | images | An empty folder shows a full magenta screen, and stays that way after images are added — VERIFIED |
| P2-16 | P2 | images | The picture jumps at every slide change — VERIFIED |
| P2-17 | P2 | images | `[images] max_image_edge` is documented in a section the code does not read — by reading |

Plus 26 P3 items.

## P1

### P1-1. A damaged playlist store is wiped — VERIFIED

`playlists.py:246-274` catches any load error and starts empty, keeping no
copy of the unreadable file.  `save()` (`:276-319`) merges with the file on
disk, skips the merge when that file will not parse, and writes the empty
state over it.  `save()` runs on every track change (`note_played`), every
heart click and every crate change.

Reproduced with a store holding one playlist and one favorite, truncated to
half its length:

```
load of truncated file -> names []
after one track change: playlists on disk {} favorites [] | backup files: []
```

The store also holds the session settings and every measured auto-level
gain.  The write itself is atomic (temp file plus replace) but is not
flushed to disk first, so a zero-length file after a power loss or a full
disk is the realistic way in.  A zero-length file, a file with a different
top-level shape and a file from a newer schema version are all overwritten
the same way.

### P1-2. Unplayable tracks cause a runaway — VERIFIED

`update()` treats "was playing and the backend reports ended or error" as
end of track and advances (`media_controller.py:1506-1519`).  If the next
track also fails, that repeats on the next frame, with no back-off and no
limit.  Each start also logs a line, writes the recently-played list to disk
(`:2477`) and, with auto-level on, starts a thread that decodes the whole
file (`:1305`).

With a backend that reports every track as ended, 120 frames (2 seconds)
produced **121 track starts, 121 store writes and 121 live measurement
threads**.  With `repeat = all` it never stops.

Ways to get there: the music drive is unmounted or asleep, files were moved
since the scan, or the audio output is unavailable.  If the files are
readable but cannot be played, each of those threads holds a decoded file in
memory (P2-7), so memory climbs by gigabytes within seconds.

## P2

### P2-1. Deleting a file or saving tags scrambles every index — VERIFIED

`delete_track()` (`media_controller.py:2005-2018`) and the tag editor's save
(`ui.py:853-855`) call `rescan_library()`.  At startup the library is
installed shuffled (`:962`, `:1991-1994`); a rescan installs it in scan
order.  Nothing remaps `_index`, `_queue` or `_played`, or the deck's
selection.

With track 7 playing and track 12 queued, deleting a third track:

```
current path matches playing: False | queued entry still the queued file: False
path published on the bus is the playing file: False (track00022.mp3 vs track00024.mp3)
```

Consequences while the old track keeps sounding:

- `publish_track_path` sends the **wrong file** to the bus that the training
  kit uses for offline analysis.
- `_publish_track_bpm` reads the wrong file's BPM tag and primes Auto VJ's
  tempo with it.
- The queue plays different tracks than were queued; next/previous move from
  the wrong place; the saved "last track" is wrong.

The rescan also runs on the main thread: 198 ms with a warm cache for
10,800 files here, more on a cold disk.  If the startup scan is still in
flight it can block for up to 300 s (`:1938`).

### P2-2. Boot playlist and last track are dropped at startup — VERIFIED

`__init__` starts the library load without waiting (`:962`), then runs
`_restore_prefs()` and `_maybe_boot_playlist()` (`:1083`, `:1096`) against a
library that is still empty.  The last-track lookup finds nothing, and the
boot playlist is judged "empty" and abandoned (`:1126-1132`).  Nothing
re-applies either when the library lands (`:1951-1976`).

| Startup | `boot_playlist = "Train"` | Last track restored |
|---|---|---|
| Scan already finished | playlists mode, 2 tracks | yes |
| Scan still running (normal) | **library mode, 30 tracks** | **no** |

For an unattended training run this means the wrong material plays, with one
warning in the log.

### P2-3. The deck window redraws from the whole library every frame — VERIFIED

Each deck frame calls `buckets()`, `view_stats()`, `view_parts()` and the
view query (`ui.py:1775`, `:1682`, `:1960`, `:1939`), and each of those
rebuilds its answer from every track.  Nothing is cached between frames.

Controller query cost at 10,800 tracks and 120 crates (measured directly):

| Call | ms |
|---|---|
| `buckets()` | 204 |
| `view_stats()` | 19.1 |
| `view_parts()` | 17.8 |
| `view_indices()`, unsorted / sorted by artist | 12.5 / 26.5 |

Whole deck frame (sub-agent's measurement, 980×640):

| Library | ms per frame |
|---|---|
| 20 tracks | 30 |
| 2,000 | 65 |
| 10,000, 60 crates | 206 |
| 10,000, 600 top-level crates | 979 |

The window's target is 50 ms.  At 10,000 tracks it draws at about 4 fps, and
because its flyout hit-areas are unset during a render (`ui.py:1261`), wheel
events over a flyout scroll the track list instead for most of that time.
The work is pure Python on the deck's own thread, so it competes with the
main render loop for the interpreter; the effect on the main loop was not
measured.  The code comments put the owner's library at about 10,800 tracks.

### P2-4. Deleted and renamed playlists come back — VERIFIED

The merge in `save()` (`playlists.py:286-296`) re-adds any playlist that is
on disk but not in memory, which is exactly the state a delete or rename
leaves.

```
delete: memory ['Keep'] | after restart ['Keep', 'Gone']
rename: memory ['Kept'] | after restart ['Kept', 'Keep', 'Gone']
```

### P2-5. The tag editor collapses multi-value tags — VERIFIED

`read_tags()` keeps only the first value of each field (`tags.py:78-81`),
and the editor writes all five fields back whether edited or not
(`ui.py:849`, `tags.py:149-153`).  Editing only the title of a file tagged
artist "Alice; Bob", genre "House; Techno":

```
after editing only the title: artist ['Alice'] genre ['House']
```

The sub-agent also verified that an ID3 v2.3 file is rewritten as v2.4, and
that a save with no edits still rewrites the file.

### P2-6. Short tracks are skipped when crossfade is on — VERIFIED

The early advance fires when the time remaining is within the crossfade
length (`media_controller.py:1494-1505`).  A track shorter than the
crossfade meets that on its first frame.  With 2 s tracks and the default
3 s crossfade: 30 track starts in 30 frames.  Jingles, stings and drops are
the realistic case.

### P2-7. Auto-level decodes the whole file into memory — VERIFIED

`measure_gain()` (`:224-249`) loads the entire track, converts it to 64-bit,
and filters it twice.

| Track | Time | Peak memory |
|---|---|---|
| 4 min | 1.6 s | +211 MB |
| 20 min | 4.8 s | +1,180 MB |

That is about 59 MB per minute, so a one-hour mix needs roughly 3.5 GB.
There is no limit on how many run at once: skipping quickly through tracks
starts one thread per track.  The thread then calls into the VLC backend and
the playlist store from outside the main thread, against the class's stated
threading contract (`:762-764`).

### P2-8. Text entry in the deck is ASCII-lowercase plus letters — VERIFIED

The tag editor and rename prompt map keys by hand (`ui.py:836-840`,
`:956-960`): Shift upper-cases letters only, Caps Lock is ignored, Ctrl+V
types a `v`, and there is no text-input event handling, so nothing outside
ASCII can be entered.  `( ) ! _ & : " ?` cannot be typed, which rules out a
title such as "Track (Extended Mix)".

### P2-9. Selection and input faults in the deck — VERIFIED

- **Search box swallows hotkeys** (`ui.py:455`, `:510-520`): with the search
  box focused, Ctrl+Alt+M and Shift+L are typed into the query and never
  reach the global handler.
- **Sort keeps the row number** (`:683-685`): the selection stays on the
  same row, so after a sort Enter plays a different track.  The same applies
  to typing in search, unhearting in Favorites and the Recent list
  reordering.
- **Clicks pass through flyouts** (`:627-640`): a double-click on a
  non-row part of an open flyout plays the track row underneath.
- **Sideways wheel scrolls down** (`:407`): a horizontal-only wheel event
  moves the list by one row.

### P2-10. "Prune missing" has no confirm — by reading

`ui.py:765-772` offers "Prune N missing" when playlist entries are not on
disk, and `:1116-1118` runs it on one click.  File deletion in the same
menu requires a second "SURE?" click.  With the music drive unmounted every
entry counts as missing, so one click empties the playlist, and the change
is saved.

### P2-11. The `aplay` fallback plays noise — by reading

`_FALLBACK_PLAYERS` is `mpv`, `ffplay`, `aplay` (`media_controller.py:55`),
and anything that is not `mpv` or `ffplay` is launched as `<player> <file>`
(`:662-663`).  `aplay` treats a file it does not recognize as raw 8-bit
audio, so an MP3 or FLAC plays as loud static.  It is reached only when
python-vlc, mpv and ffplay are all unavailable; `aplay` itself is present on
almost every Linux system.  Not run, since that would mean playing audio.

### P2-12. Past 128 images, whole folders vanish — VERIFIED

The pack loader takes the first 128 files alphabetically across the whole
directory tree (`image_showcase.py:43-51`, `:369-386`), but each activation
picks one folder at random from **all** folders on disk (`:427-434`).  A
folder with no loaded images falls back to "everything loaded".

Three folders of 100 images each: 128 loaded (100 from `a`, 28 from `b`,
none from `c`).  Over 40 activations: `a` alone 12 times, `b` alone 13
times, and a mix of `a` and `b` 15 times.  Folder `c` was never shown, and
72 of `b`'s images never were either.

### P2-13. Photo orientation is ignored — VERIFIED

`_decode_capped()` (`:68-73`) does not apply the EXIF orientation tag.  A
400×200 file marked "rotate 90°" decodes as 400×200; a viewer that honors
the tag shows 200×400.  Portrait photos from a phone or camera appear on
their side.

### P2-14. First activation decodes on the render thread — VERIFIED

When `preload_images` is off, or the effect is activated before the
background prefetch finishes, `_init` decodes and uploads the whole pack
synchronously (`:365-397`).  Twelve 12-megapixel JPEGs took **3.4 s**
(287 ms each); a full 128-image pack would be about 37 s of frozen picture.
The shipped config has preload on, which reduces this to a 16 ms upload per
image at startup.

Related sizes: a 2048-pixel image is 9.4 MB, so a full pack is 1.2 GB of
decoded bytes held in RAM until first use and 1.6 GB of texture memory with
mipmaps, kept for the life of the process.  At the maximum allowed
`max_image_edge` of 8192 the texture figure is about 26 GB.

### P2-15. An empty folder shows magenta, permanently — VERIFIED

With no images, the pack is a single 1×1 magenta texture (`:374-380`).
Rendered full-screen: center pixel (252, 0, 254).  That placeholder is
cached for the life of the process, so adding images to the folder does not
help until the app is restarted.  The same cache means new images in any
folder, and a changed `max_image_edge`, are ignored until restart.

### P2-16. The picture jumps at every slide change — VERIFIED

Same cause as Video Clips (the two effects share the shader): the incoming
slide moves from slot B to slot A and takes slot A's pan and zoom at once.
With an identical still in every slot, the normal frame-to-frame change is
1.4–1.7 (out of 255) and the change on the slide-advance frame is 22–26, in
five of five trials.

### P2-17. `max_image_edge` is documented in the wrong section — by reading

The README changelog and the function docstring (`:54-60`) name the key as
`[images] max_image_edge`.  The code reads it from the effect's own config,
which core fills from `[effects.ImageShowcase]` (`app.py:6196`, `:6322`).  A
value set under `[images]` is never seen.

## P3

### media-01 controller

| ID | Where | Finding |
|---|---|---|
| P3-1 | `media_controller.py:888`, `:895-897`, `:1068-1069` | The `x or default` pattern turns a configured 0 into the default: `volume = 0` becomes 100, `now_playing_banner_delay_s = 0` becomes 2.0, `crossfade_ms = 0` becomes 3 s — VERIFIED |
| P3-2 | `:2457` | The track id is the file name only; two files named `01 Intro.mp3` share an id — VERIFIED.  A late auto-level result can be applied to the other one, and consumers watching for a track change may miss it |
| P3-3 | `:424` | `--aout=pulse` is passed to VLC on every platform.  On Windows there is no such output — SUSPECTED silent playback; not testable here |
| P3-4 | `:672-689` | The subprocess backend pauses with SIGSTOP, which does not exist on Windows; the failure is swallowed and the player is marked paused while it keeps playing.  By reading |
| P3-5 | `:1663` | The BPM tag is read on the main thread at each track change.  With 15 MB of embedded cover art that is 46 ms and 63 MB — VERIFIED by the sub-agent |
| P3-6 | `:2397` | `queue_len` in the snapshot is "tracks after this index", which means nothing under shuffle, a playlist or a filter |
| P3-7 | `:1208-1222`, `:2311` | Session settings are saved only at clean shutdown |
| P3-8 | `:16-18`, `:762-764`, `:1680-1683` | Stale docstrings: crossfade is described as not implemented; the class claims no background threads; `handle_key` lists hotkeys (project rule: `HELP_ENTRIES` only) |
| P3-9 | `:1296-1297`, `:2216`, `:2478-2479` and others | Bare `except: pass` hides store-write and playback failures that the operator would want to know about |

### media-01 deck window, store and tags (sub-agent; VERIFIED unless noted)

| ID | Where | Finding |
|---|---|---|
| P3-10 | `ui.py:1085-1091` | "Move up/down" shifts the selection even when the visible order did not change (a dead entry between two live ones, or a sorted view) |
| P3-11 | `ui.py:420-423` | The pointer-left-window branch is unreachable, so hover and tooltips never clear |
| P3-12 | `ui.py:457-462` | Esc with a menu or flyout open arms window-close; a second Esc closes the deck |
| P3-13 | `ui.py:406-408`, `:424-429` | Right-click and wheel still act on the list underneath an open rename or tag prompt |
| P3-14 | `ui.py:463-488` | Deck keys ignore modifiers: Ctrl+Alt+Space plays/pauses, Alt+Enter plays the selection |
| P3-15 | `ui.py:569-581` | The list scrollbar wins over a flyout's scrollbar where they overlap (7 px) |
| P3-16 | `ui.py:668-674`, `:715-718` | Switching tab, crate or playlist resets the selection but not the scroll position, leaving the selection off-screen |
| P3-17 | `ui.py:2559-2572` | A NaN spectrum band makes the deck's render raise every frame (frozen deck, one warning per frame).  Whether the analyzer can emit NaN is unconfirmed |
| P3-18 | `ui.py:747`/`:1939`, `:2168-2185` | Two one-frame races between the deck thread and the main thread (library swap; menu cleared mid-draw).  Second one by reading |
| P3-19 | `ui.py:765` vs `:1921` | The row's "missing" badge counts entries not in the library; the Prune item counts entries not on disk.  They disagree during a scan and for files outside the media folder.  By reading |
| P3-20 | `playlists.py:276-319` | Every save re-reads, parses and rewrites the whole store on the main thread: 54 ms at 11,000 gains and 20 playlists of 500.  Gains and favorites for deleted files are never pruned |
| P3-21 | `playlists.py:316-319` | `save()` returns False on failure and every caller ignores it; the deck shows success on a read-only folder |
| P3-22 | `tags.py:83`, `:136`, `:130`, `:109` | Log lines say `dj-mixer-01`; a `None` field is written as the text "None"; BPM values like `128,5` or `128 BPM` read as 0 |
| P3-23 | `ui.py`, `README.md:276`, `docs/operations.md:82` | Deck-local keys are documented in docstrings, tooltips, the README and the docs, and are missing from `HELP_ENTRIES`.  British spellings in comments in `ui.py` and `tags.py` |

### images-01

| ID | Where | Finding |
|---|---|---|
| P3-24 | `image_showcase.py:467-468` | A beat adds 0.12 to the crossfade for one frame, then it snaps back (0.125 → 0.249 → 0.133) — VERIFIED.  Same as Video Clips |
| P3-25 | `:286-313` | Two `prefetch_async()` calls before the first finishes start two decode threads — VERIFIED |
| P3-26 | `:100-124`, `:486-503`, `:31-34` | `_StylePreset` carries nine fields the shader never receives (only the index is sent); `render()` builds a closure and tuples every frame; the caches are module-level globals; no tests in the drop-in |

## Recommended tests

Each is written to fail against the current code where a finding is open.
All are pure Python unless marked GL.

### media-01

| Test | Covers |
|---|---|
| A truncated, zero-length or wrong-shaped store file is never overwritten without a backup copy, and the app says so | P1-1 |
| A backend that reports every track as failed stops after a bounded number of skips, with at most one measurement thread alive | P1-2 |
| After `delete_track()` and after a tag save, the current index, every queued entry and the published path still refer to the same files as before | P2-1 |
| A controller built while the prescan is still running ends up in the boot playlist, and restores the last track, once the library lands | P2-2 |
| One deck frame at 10,000 tracks calls each whole-library query at most once, and none when nothing changed since the last frame | P2-3 |
| Delete then restart, and rename then restart, leave only the expected names | P2-4 |
| Editing one field in the tag editor leaves multi-value artist and genre tags intact; a no-edit save does not touch the file | P2-5 |
| A track shorter than the crossfade plays to its end | P2-6 |
| `measure_gain()` on a 20-minute file stays under a fixed memory bound; rapid skipping never runs more than one measurement at a time | P2-7 |
| The text prompts accept `(`, `)`, `&`, `_`, capitals via Caps Lock, pasted text and a non-ASCII character | P2-8 |
| With the search box focused, Ctrl+Alt+M still reaches the global handler | P2-9 |
| After a sort, the selected track is the same track | P2-9 |
| A click inside an open flyout never reaches the row beneath | P2-9 |
| Prune requires a confirm, and refuses when every entry is missing | P2-10 |
| The fallback player list never selects a player that cannot decode the file type | P2-11 |
| `volume = 0`, `now_playing_banner_delay_s = 0` and `crossfade_ms = 0` are honored | P3-1 |
| Two files with the same name in different folders get different track ids | P3-2 |

### images-01 (no tests in the drop-in today)

| Test | Covers | Kind |
|---|---|---|
| With three folders of 100 images, every activation shows images from exactly one folder, and every folder is reachable | P2-12 | Python (fake context) |
| A JPEG with EXIF orientation 6 is decoded upright | P2-13 | Python |
| `_init` with no prefetch does not decode more than a bounded number of images before returning | P2-14 | Python (fake context) |
| An empty folder renders black (or the effect reports itself unavailable), and images added later are picked up at the next activation | P2-15 | GL |
| With the same still in every slot, the frame before and after a slide change differs by no more than a normal frame step | P2-16 | GL |
| `max_image_edge` set where the README says takes effect | P2-17 | Python |
| A beat does not change the crossfade for one frame and revert | P3-24 | Python |
| Two back-to-back `prefetch_async()` calls start one thread | P3-25 | Python |
| A corrupt image, a zero-length file and an oversized image are each skipped with a warning | holds today; unpinned | Python |

## What works

- **Tag cache.**  A warm scan of 10,800 files takes 198 ms against 4.6 s
  cold; it is keyed per media folder and written atomically.
- **Startup.**  The library scan and the image decode both run on background
  threads during the splash; a play request during the scan is honored when
  the library lands.
- **VLC volume guard.**  Volume is never set on a player without a live
  output, the documented crash path, and auto-level goes through the same
  guard.
- **Shuffle bag, repeat modes and the up-next queue** behave as documented
  under the existing 174 tests.
- **Playlist names** are only JSON keys: path separators, `..`, unicode and
  a 5,000-character name create no extra files.
- **Tag module on bad input.**  Empty, truncated, random and missing files
  return defaults or a clear failure; WAV and AIFF writes are refused.
- **Image safety caps.**  Oversized images are downscaled at decode and the
  pack is capped; corrupt files are skipped with a warning.
- **Independence.**  No `print()`, no private `app` access; both drop-ins
  go through `vj_api`.

## What this audit did not do

- Play audio, start VLC, or open the deck window.  Backend behavior is from
  reading plus a fake backend; real SDL input behavior was not exercised.
- Read the owner's library, playlists or images.
- Measure the deck window's effect on the main render loop.
- Test Windows.
- Fix anything.  The bandit findings are reported only.
