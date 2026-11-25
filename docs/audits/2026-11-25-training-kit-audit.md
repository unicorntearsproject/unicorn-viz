# Training kit audit — `drop-ins/training-kit-01` (2026-11-25)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-25

Scope: the `training-kit-01` drop-in at submodule commit `3322289`
(`0.42.13`, about 13,000 lines): the two modules the live app loads
(`keystroke_logger.py`, `corpus_writers.py`) and the offline tools — the
packager (`tools/package_training_set.py`, 4,498 lines), the headless daemon,
the replay harness, the scorecard and placement tools, and the
train/promote pipeline.

**Tuning was out of scope.**  No weight, threshold or rubric number is judged.

Method:

- Read in full: both runtime modules, the packager, `director_placement.py`,
  `session_replay.py`, `track_replay.py`, `training_daemon.py`, everything in
  `tools/training/`, `promote_weights.py`,
  `generate_training_set_description.py`, and nine smaller tools.
- **Not read line by line** (pattern search and `--help` only):
  `recommender_fingerprints.py`, `bpm_eval.py`, `essentia_cache.py`,
  `genre_report.py`, `label_genres.py`, `library_feature_scan.py`.  There are
  no line-level findings for these six.
- No network call was made and no LLM was contacted; provider keys were
  unset for every command.  The owner's real `logs/` and `assets/training/`
  were never read.
- The packager, writers and replay were exercised on synthetic fixtures in a
  fake repository under `/var/tmp/uv-dropin-audit/kit/`.  The daemon was not
  run; its signal handling was reproduced with a stand-alone copy of the same
  pattern.
- **VERIFIED** = reproduced on a fixture.  **SUSPECTED** marks what could not
  be confirmed.

Severity: **P1** can lose, misplace or corrupt the owner's data, leak
private input, or crash the live app.  **P2** training data, scorecards or
LLM scores silently wrong, or an experiment tool that silently does nothing.
**P3** latent, cosmetic or a broken minor tool.

---

## Summary

| ID | Sev | Area | One line |
|---|---|---|---|
| P1-1 | P1 | packager | Every packaging run, including each replay's auto-package, sweeps the real `screenshots/` and `recordings/` into the bucket — **VERIFIED** |
| P2-1 | P2 | packager | No rollback: a failure after the first move strands a bucket with no scorecard and no record — **VERIFIED** |
| P2-2 | P2 | daemon | After Ctrl+C or SIGTERM the daemon packages while the app is still running — **VERIFIED** (pattern) |
| P2-3 | P2 | packager | Live and sequence corpora are picked independently; two sessions can be mixed in one bucket — **VERIFIED** |
| P2-4 | P2 | LLM prompt | The prompt states three recommender weights that contradict the live weights in the same prompt — **VERIFIED** |
| P2-5 | P2 | LLM prompt | The prompt says no genre tag is included; the payload includes one — **VERIFIED** |
| P2-6 | P2 | LLM prompt | "Current" constants are code defaults, not what the session ran; a failed reader falls back silently — **VERIFIED** |
| P2-7 | P2 | replay | `--override` is a silent no-op for most director keys, and typos are accepted — **VERIFIED** |
| P2-8 | P2 | replay | `--seed` does not make replays reproducible: 453 of 940 effect picks differ — **VERIFIED** |
| P2-9 | P2 | training | Train-then-promote cannot change runtime behavior, and accepts one-row and NaN fits — **VERIFIED** |
| P2-10 | P2 | privacy | The keystroke logger stores reconstructable typed text in plaintext, and packaging archives it — **VERIFIED** |
| P3-1…P3-20 | P3 | various | See the P3 table |

---

## P1

### P1-1. Packaging sweeps the real screenshots and recordings — VERIFIED

`tools/package_training_set.py:4353-4371`; called by
`tools/session_replay.py:611-623`.

The packager moves every file under `<repo>/screenshots` and
`<repo>/recordings` into the bucket.  Those two source directories are fixed:
`--corpus-dir`, `--logs-dir`, `--sets-root` and `--training-root` do not
redirect them.

A replay auto-packages when it exits (the default; replays are often run four
at a time).  Each one therefore moves the owner's live-show screenshots and
recordings into `assets/training/accelerated/<playlist>/<NNN>/`.  The files
are not deleted, but they leave the place the owner expects them, attached to
a replay that has nothing to do with them.  Subfolders are flattened, so two
files with the same name stop the run partway (see P2-1).

Fixture run with all four path flags pointed at a sandbox:
`screenshots/unicornviz_real.png`, `screenshots/sub/x.png` and
`recordings/owner_show.mkv` left the fake repository and landed in the
sandbox bucket.

Worth checking by hand: whether past replay buckets under
`assets/training/accelerated/` already hold live recordings.  This audit did
not look at that directory.

**Fix:** add `--screenshots-dir` / `--recordings-dir`; skip both when
`--logs-dir` is overridden; have the replay packager pass none.

---

## P2

### P2-1. A failure midway strands a bucket — VERIFIED

`package_training_set.py:4314-4371`, `4380-4381`, `1385-1390`.  Corpora are
moved first; the log sweep, media sweep, scorecard and notes prompt follow.
An exception after the first move leaves a bucket with no `scorecard.md` and
no `SESSION_TRAINING_LOG.md` line, and a re-run says "Could not find both
live and sequence corpus files".

Two ways to hit it, both reproduced:

- `logs/a/b.log` and `logs/a__b.log` flatten to the same name
  (`FileExistsError`); the bucket held the corpora and one log, the rest
  stayed in `logs/`.
- stdin closed without `--no-prompt`: `EOFError` at the notes prompt, after
  everything had moved.

**Fix:** do all prompting before any move; write a manifest of intended moves
first; catch failures and report what moved.

### P2-2. The daemon packages a running session — VERIFIED (pattern)

`tools/training_daemon.py:149-155`, `345-356`, `731-740`.  The signal handler
raises `SystemExit`, which the `except (KeyboardInterrupt, SystemExit): pass`
around `uviz_proc.wait()` swallows.  The app was started in its own session,
so it never got the signal.  The packager then runs against a live app, which
is only killed later by the exit cleanup.

Rows written after the scorecard are not counted.  After the corpus file has
been moved, the app's next write re-creates the whole live corpus at the old
path, leaving a stale duplicate for a later bucket to pick up (reproduced).

**Fix:** on a signal, terminate the app and wait for it before packaging.

### P2-3. Corpora are paired by "newest", not by session — VERIFIED

`package_training_set.py:4263-4264`, `1375-1382`, `4339-4351`.  The live and
sequence corpora are each chosen as the newest file by mtime, with no check
that their timestamps match.  Session A (both files, 20 Nov) plus session B
(live only, 24 Nov) produced one bucket holding B's live corpus and A's
sequence corpus, with no warning, and left A's live corpus orphaned.  Every
file in `logs/` goes into whichever bucket is being built.

**Fix:** pair by filename stamp; refuse or warn on a mismatch; list leftovers.

### P2-4. The prompt contradicts its own weights — VERIFIED

`package_training_set.py:3122-3128`.  Fixed text tells the model
`spectral_shape_fit (weight 1.0)`, `kick_regularity_fit (weight 0.5)` and
`top_cand_fit (weight 0.4)`.  The live weights line in the same prompt reads
0.191, 0.459 and 0.  The model is asked to reason about recommender behavior
from two different sets of numbers.

**Fix:** format those three from the loaded dict, or delete the paragraph.

### P2-5. Genre tag: stated absent, actually present — VERIFIED

`package_training_set.py:3105-3109` vs `2382`.  The prompt says "No library
genre tag is included in this payload"; `live_corpus_snapshot` carries
`"track_genre": "House"`.  The scorer is told its judgment is tag-free when
it is not.

**Fix:** drop the field from the snapshot or correct the text.

### P2-6. "Current" constants are defaults — VERIFIED (fallback)

- `package_training_set.py:512`, `775-779`: the live readers build a bare
  tracker with `{}` and a bare controller pinned to the `normie` preset.
  They report code defaults.  Anything the session actually ran with from
  `config.toml` or from another mood preset is invisible.  (The project's own
  notes record that the tuned v3 state lives only in `config.toml`.)  By
  reading.
- `2994-2996`, `527-529`, `661-663`, `801-803`: when a reader raises, the
  whole fallback dict is used with only a debug log and no marker in the
  prompt.  With the repo root unreachable the prompt read
  `_BPM_LOCK_RELEASE_CONFIDENCE = 0.25` and `drop_trigger_threshold = 0.6`;
  the live values are 0.2 and 0.65.

**Fix:** stamp the effective values into the corpus at session start and read
those; mark fallback values in the prompt and warn.

### P2-7. Replay overrides that do nothing — VERIFIED

`tools/session_replay.py:297`, `674-683`; cause in
`drop-ins/auto-vj-01/auto_vj.py:3750-3756`, `2804-2815`.  The controller
reads most director keys from config only when the USER mood is active.
Replay runs in auto mood, so those overrides are ignored.  Unknown keys are
accepted without a warning.

On a 400-second synthetic three-track session (baseline identical across
three runs):

| Override | Effect on detector / director / recommender output |
|---|---|
| `build_min_hold_s=30` | none; changes output only with `auto_profile_enabled=false` added |
| `drop_trigger_threshold=0.99`, `drop_cooldown_s=600` | none, even with `auto_profile_enabled=false` (cause not isolated) |
| `phrase_bias_max=0.0`, `tempo_hold_s=30`, `phrase_tolerance=0.3` | none (cause not isolated) |
| `drop_triger_threshold=0.99` (typo), `_V2_DWELL_BARS=1` | none, no warning |
| `beat_tracker_engine=v2`, `tactus_preference_ratio=1.5`, `mode_snap_downbeat=0`, `drop_bass_delta_min=0`, `genre_matcher_endorse_sigma=0.5`, `profile_auto_reco_switch_backoff_mult=10`, `profile=chill` | changed |

**Consequence:** any past replay experiment that overrode a key in the first
three rows, and concluded "no difference", measured nothing.  Before trusting
a null from the replay harness, confirm the key is one that takes effect.
This is the same defect as Auto VJ audit P2-1, seen from the tool side.

**Fix:** reject unknown keys; print each override's resolved effective value
after the controller is built.

### P2-8. `--seed` does not seed the controller — VERIFIED

`tools/session_replay.py:234-236`, `660-661`;
`drop-ins/auto-vj-01/auto_vj.py:3011`.  The controller uses its own
`random.Random()` seeded from the OS.  Three runs with `--seed 1` had
identical detector, mode and recommender columns, but `effect_name` differed
in 453 of 940 rows.  The stub's effect pick also depends on the count of all
recorded calls.

**Fix:** pass a seeded RNG into the controller.

### P2-9. Train-then-promote changes nothing, and accepts bad fits — VERIFIED

`tools/training/train_weights.py:55-77`, `tools/training/training_lib.py:39-54`,
`613-676`, `tools/promote_weights.py:41-55`; loader at
`drop-ins/auto-vj-01/auto_vj.py:1858-1880`.

- The fit is keyed by audio feature columns (`bpm`, `rms`, …).  The runtime
  loader accepts only recommender term names (`zcr_fit`, …).  The
  intersection is empty, yet the tool prints "Promoted … Takes effect on the
  next Auto VJ startup".
- A one-row corpus and a corpus with a NaN target both fit with exit 0 and no
  warning (`rows 1.0 r2 0.0`; `bpm w nan bias nan`), and `promote_weights.py`
  promoted the NaN file.

**Fix:** validate keys against `_DEFAULT_RECO_WEIGHTS`; reject non-finite
values; require a minimum row count.

### P2-10. The keystroke log holds typed text — VERIFIED

`keystroke_logger.py:55-88`; call site `unicornviz/hotkeys.py:715-728`.  The
logger runs at the top of the hotkey handler, before any modal routing, and
records every keydown that reaches it with its modifiers, in plaintext, one
JSON line per key, flushed per event, under `logs/keystrokes-<stamp>.log`.

- Text typed into the banner editor, the CTA editor and the search boxes can
  be read back (a logged sequence reads back as `HUNTER2`).
- Not captured, by reading `unicornviz/app.py:6833-6919`: config-editor text
  including the secret rows (stream key), profile names and hotkey capture.
  Those are consumed before the handler.
- Nothing from the file is sent to an LLM.
- Packaging moves the file into the bucket, so it is archived with the
  training set.
- It is off by default in code; `config.toml` in this checkout enables it.

**Fix:** suppress logging while a text-input mode is active.

---

## P3

| ID | Where | What | Fix |
|---|---|---|---|
| P3-1 | `package_training_set.py:1421-1424`; `director_placement.py:106-109` and the other loaders | Torn or invalid JSONL lines are dropped with no count — **VERIFIED** | Count and print |
| P3-2 | `package_training_set.py:1684-1701` vs `2183-2196` | The scorecard's lock percentage counts keyframe rows; the console table and the LLM metadata count heartbeats only (69.2 % vs 60.0 % on one fixture) — **VERIFIED** | One denominator |
| P3-3 | `package_training_set.py:54-56` | `_BPM_LOCK_CONFIDENCE_FLOOR = 0.45` is described as mirroring the controller's lock threshold, which is 0.6; CLAUDE.md says they must stay in sync — **VERIFIED** | Read it live, or correct the comment and the rule |
| P3-4 | `package_training_set.py:1140-1186` | Prints "No tagged rows (`track_genre` empty/absent)" when rows have a genre but no recommendation — **VERIFIED** | Separate the cases |
| P3-5 | `package_training_set.py:3901-3903` | Set descriptions are never generated: it looks for the generator under the repo-root `tools/`, where it does not exist — **VERIFIED** | Point at the sibling file |
| P3-6 | `generate_training_set_description.py:214-222`, `574-599` | Fixed sets root; sends each track's artist and title to `itunes.apple.com` unless `--no-itunes-lookup` is passed (not run) | Path flags; make the lookup opt-in |
| P3-7 | `package_training_set.py:4314-4315`, `3957` | `--force-regen-detector-score` does nothing: `main()` always creates a new bucket | Remove, or add a re-score path |
| P3-8 | `tools/score_existing_buckets.py:31` | Dead: `AttributeError … '_score_detector_with_llm'`.  CLAUDE.md and `docs/adr/training-model.md:510` still name that function — **VERIFIED** | Fix or delete; update both docs |
| P3-9 | `corpus_writers.py:219-241` | On a persistent write failure the live corpus writer retries a full rewrite every frame (600 attempts in 600 frames) and logs at debug.  `mkdir` is outside the `try`, so `upsert` and `shutdown()` can raise; the unguarded call at `auto_vj.py:5692` then skips the sequence writer's shutdown — **VERIFIED** | Back off; warn once; move `mkdir` inside |
| P3-10 | `tools/promote_weights.py:65-67` | The archive name has one-second resolution and `rename` overwrites: two promotions in one second lost the original weights — **VERIFIED** | Uniqueness suffix; refuse to overwrite |
| P3-11 | `tools/session_replay.py:368-438` | Replay fidelity: `position_s` jumps back 10 s into every crossfaded track; reported audio length counts the overlap twice (421.9 vs 401.8 s); `track_id` is the file stem, so same-named files merge; the stub never simulates projectM, section hints, session info or video opacity — **VERIFIED** for the first two | Fix the clock; qualify the id; document the gaps |
| P3-12 | `package_training_set.py:3419-3432`, `4027`, `3475-3488` | LLM response handling: a JSON array crashes after files have moved (Anthropic path); `{}` counts as success and writes score files full of `n/a` with no failure marker; no failover when the first provider's call fails; the Anthropic call sets no timeout — **VERIFIED** for the first two | Validate the schema; check the stop reason; set a timeout |
| P3-13 | `package_training_set.py:3276-3280`, `3380`, `3389` | Stale prompt text: still asks for a centroid comparison and lists `band_fit`, both retired — **VERIFIED** | Remove |
| P3-14 | packager prompt assembly | Track titles, artists and the `--set-description` file are inserted verbatim into the prompt.  Output is advisory and never auto-applied — **VERIFIED** | Delimit and label as data |
| P3-15 | `package_training_set.py:4294`, `4306` | `--set-name ../../x` creates directories outside the sets root | Sanitize |
| P3-16 | `package_training_set.py:4378`, `1820`; `generate_training_set_description.py:402` | Dates are packaging dates, not session dates | Use the corpus stamp |
| P3-17 | `tools/training/sync_corpus_from_logs.py:300`; `training_lib.py:235`, `326` | Never incremental (the output path is re-stamped each run) and only sees Spotify IDs: mixer, media and replay sessions yield zero tracks | Stable path; accept the other ids |
| P3-18 | `corpus_writers.py:115-117`; `keystroke_logger.py:85-86` | Encode, write and flush run synchronously on the render thread per row or key; the keystroke logger swallows all I/O errors | Buffer; log once |
| P3-19 | `package_training_set.py:1365-1372`, `2859-2867` | Bucket numbering reuses gaps, so a removed `002` is refilled out of order; the `.env` loader exports every key in the file to child processes, not only the two provider keys | Next = max + 1; export two keys |
| P3-20 | `tools/gen_bpm_eval_corpus.py`, `analyze_autovj_log.py`, `own_wins_family_scorer.py` | No argument parser: `--help` is taken as an output directory (12 files written to `./--help/` during this audit), as a log path, or ignored in favor of a hard-coded bucket | Add parsers |

---

## What leaves the machine

**Scoring call** (one user message), in order:

1. Fixed rubric text, plus the `--set-description` file if given.
2. Detector constants.
3. Detector payload: set and bucket names, row and song counts, a per-track
   snapshot (title, artist, BPM, confidence, genre tag, mood profile),
   wall-clock start and end, BPM/confidence/lock statistics, band means,
   engine versions, per-song medians with mixer and Essentia BPM and key.
4. Recommender payload: counters, profile distributions, up to 30 switch
   events with timestamps.
5. Director constants, then the director payload: counts, mode and profile
   distributions, placement metrics, up to 50 sampled events.
6. Recommender weights and the profile expected-values table.
7. The response schema.

A second call sends the full `scorecard.md`, the first call's result and the
report file names.

- **Not sent:** file paths, hostname, username, keystroke logs, app logs,
  screenshots.  For media and replay sessions the track id and title are the
  file name.
- **Models as written:** `gpt-5.6-terra` (packager, chat completions),
  `claude-opus-4-8` (packager, streaming), `gpt-5.3-codex`
  (`generate_training_set_description.py`).
- **Keys** come from the environment or the repo-root `.env`, go only to the
  SDK client or the Authorization header, and are not logged.  A provider's
  exception text is written to `LLM_SCORING_FAILED.txt`; whether that text
  can contain a key fragment is not confirmed.
- **Other hosts:** the iTunes lookup (P3-6).

---

## Payload constants vs live code

All three live readers succeeded at this commit (detector 75 of 75 names,
director 70 of 70, recommender 13).  Fallback values that differ:

| Dict | Name | Fallback | Live |
|---|---|---|---|
| `_DETECTOR_CONSTANT_DEFAULTS` | `_BPM_LOCK_RELEASE_CONFIDENCE` | 0.25 | 0.2 |
| `_DETECTOR_CONSTANT_DEFAULTS` | `_V2_DWELL_BARS` | 8.0 | 32.0 |
| `_DIRECTOR_CONSTANT_DEFAULTS` | `drop_trigger_threshold` | 0.60 | 0.65 |
| `_DIRECTOR_CONSTANT_DEFAULTS` | `drop_trigger_fastlane` | 0.75 | 0.8125 |
| `_RECO_WEIGHT_DEFAULTS` | `band_fit` | 0.0 | key absent |

Names missing from the dicts (never sent, live reader or not): about 30
detector constants and 23 director/decider keys.  The lists are in the Auto
VJ audit of the same date, under "Sync obligations".  The comment at
`package_training_set.py:141-142` says string selectors are left out because
the reader casts to float; the reader is now type-aware, so that reason no
longer holds.

---

## Rule violations and design debt

- **Required docs missing:** no `docs/operations.md`, `configuration.md`,
  `integration.md` or `troubleshooting.md` (Documentation SOP for complex
  drop-ins).
- **"Packaging must move all files under `logs/`":** the sweep excludes
  `logs/replay`.  Deliberate and documented in the README; the rule text was
  not updated.
- **No `tests/` directory** in the drop-in; its tests live in the main repo.
- Two version constants: `__init__.__version__ = '0.42.13'` (matches the
  README) and `corpus_writers.__version__ = '0.7.1'`.
- Hard-coded owner paths and session names:
  `decision_log_to_sequence_corpus.py:113` (`/home/jj/Music/crates`),
  `own_wins_family_scorer.py`, `score_existing_buckets.py:40-47`.
- `training_daemon.py` starts the app and the packager with `python3` from
  PATH, not the checkout's venv (490, 518).  Its docstring says media
  playback is "shuffled"; the code passes the unshuffled favorites flag.
- Four tools cannot be pointed away from the repo's own data:
  `generate_training_set_description.py`, `ingest_arbitration.py`,
  `own_wins_family_scorer.py`, `score_existing_buckets.py`.
- Several docstrings still point at `drop-ins/auto-vj-01/tools/`, where these
  scripts used to live.
- Clean: no `shell=True`, `eval`, `exec`, `pickle` or `yaml.load`; every
  `subprocess` call uses a list; no `print()` in the two runtime modules; no
  hotkeys, so no help entries are owed.
- `--help` works for 21 scripts and fails for 4 (P3-8, P3-20).

---

## What this audit did not do

- No LLM call, no Essentia run, no real daemon run, no packaging against real
  session data.  Disk-full behavior was reasoned from a read-only-directory
  test.
- Six tools were not read line by line (listed under Method), nor were the
  drop-in's other docs or anything under `tools/baselines/`.  README lines
  718-920 were not read.
- Did not look inside `assets/training/` to see whether P1-1 has already
  moved recordings there.
- Did not fix anything.

Housekeeping: the 12 synthetic click-track files that `gen_bpm_eval_corpus.py
--help` wrote into the audit seat were moved to
`/var/tmp/uv-dropin-audit/kit/stray-help-dir/`; nothing else in any checkout
changed.
