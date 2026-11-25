# Tools audit — `tools/` (2026-11-25)

Owner: DJ Unicorn Tears
Status: complete — all findings open; nothing fixed in this pass
Last updated: 2026-11-25

Scope: the repo-root `tools/` directory at master `da0255a` (140 files, about
25,000 lines of bash, PowerShell and Python), plus the files those scripts
call: root `install.sh` and `run.sh`, `packaging/` (Inno Setup, Flatpak, Snap),
`requirements.txt`, `pyproject.toml`, `.pre-commit-config.yaml`.

Two parts:

- **Part A — build, packaging, install, launchers**: `tools/packaging/`
  (including `ft-patched/`), `tools/install/`, `tools/install_*.{sh,ps1,bat}`,
  `tools/launchers/`, `tools/seat_worktree.sh`, `tools/version_report.py`,
  `tools/gen_third_party_licenses.py`.
- **Part B — commit hooks, generators, profiling, beat-tracker bench**:
  everything else.

Method:

- Nothing was built, downloaded, installed, signed or released, and no
  network call was made.  Scripts were run only with `--help` / `--dry-run`
  after reading them, with `bash -n` and `shellcheck`, or as pure functions
  sourced into a scratch shell.  `fetch_acid_ans.py` was read, not run.
- PowerShell was read only (`pwsh` is not installed), so Windows findings are
  **by reading** and none was run on a Windows host.
- The three hook scripts were tested against scratch files with deliberate
  errors.  Generators were run with output redirected to scratch.
- **VERIFIED** = demonstrated safely (named script or command).
  **SUSPECTED** marks what could not be confirmed.
- The two 4,500-line frozen detector copies in the bench were diffed, not
  reviewed.  A few files were skimmed rather than read in full; see "What
  this audit did not do".

Scratch: `/var/tmp/uv-dropin-audit/{pkg,tools}/`.

Severity: **P1** can ship a tampered or wrong artifact, delete the owner's
data or release archive, damage the shared repository, or let broken code
through a commit gate in a common case.  **P2** an install or build that
fails for a realistic user, a release that can go out wrong, or a gate with a
real blind spot.  **P3** latent, cosmetic or an unused script.

---

## Summary

| ID | Sev | Area | One line |
|---|---|---|---|
| A-P1-1 | P1 | release | `release.sh` overwrites a same-version tarball and rewrites `SHA256SUMS` in place in the release archive |
| A-P1-2 | P1 | release | `build_windows_portable.sh` deletes the previous zip of the same name before writing |
| A-P1-3 | P1 | install | The one-line installer downloads and runs two scripts with no checksum, from a GitHub owner and branch that do not match this repo |
| B-P1-1 | P1 | hooks | The docs-check hook cannot fail — **VERIFIED** (40 standing errors pass) |
| A-P2-1 | P2 | seats | `seat_worktree.sh` moves a drop-in branch back to the pin, dropping unpushed commits from it — **VERIFIED** (git semantics) |
| A-P2-2 | P2 | seats | `seat_worktree.sh` does not scrub `GIT_*` |
| A-P2-3 | P2 | install | The default Linux install fails on Fedora, Debian and Ubuntu: `/etc/os-release` overwrites `VERSION` — **VERIFIED** |
| A-P2-4 | P2 | install | `install.sh` aborts on any unsigned release — **VERIFIED** |
| A-P2-5 | P2 | install | Every integrity check in `install.sh` traces back to the download host and most only warn |
| A-P2-6 | P2 | runtime | The 3.11 runtime has no pinned digest and is still the default flavor — **VERIFIED** |
| A-P2-7 | P2 | runtime | Six install and build paths still yield Python older than 3.13, which core cannot use |
| A-P2-8 | P2 | release | Local wheels are verified against a checksum file in the same folder, not the committed trust file |
| A-P2-9 | P2 | release | A release can ship a version other than `unicornviz.__version__`; the clean-tree guard covers part of one path — **VERIFIED** (`--version 9.9.9` accepted) |
| A-P2-10 | P2 | release | The Windows bundle gets no checksum or manifest entry; its bundled VLC installer is unverified |
| A-P2-11 | P2 | release | Any pip failure for a drop-in dependency in the Windows bundle becomes a warning |
| A-P2-12 | P2 | licensing | `openai` and `anthropic` ship in every bundle but are excluded from the license notice; unknown licenses pass |
| A-P2-13 | P2 | install | rpm, deb and all-users Windows installs are not user-writable, and the app writes next to itself |
| A-P2-14 | P2 | install | Arch installs run `pacman -Sy` without `-u` (partial upgrade) |
| A-P2-15 | P2 | install | Uninstall removes the whole prefix, user data included, with no prompt |
| A-P2-16 | P2 | Windows | `install_windows.ps1` prints "Install complete." after failed pip steps; `--cli` mode in the `.bat` is broken |
| A-P2-17 | P2 | Windows | The GUI launcher's Update button moves every drop-in off its pin and can hang |
| A-P2-18 | P2 | Windows | Buttons that start PowerShell break when the path contains a space |
| A-P2-19 | P2 | release | `fetch_wheelhouse.sh` truncates `SHA256SUMS` in its destination |
| A-P2-20 | P2 | release | Shipped wheels are built with unpinned tools and `:latest` base images |
| A-P2-21 | P2 | install | `uv_wheelhouse_args` breaks on a path with spaces |
| B-P2-1 | P2 | hooks | The docs check has no index-membership check and exempts `docs/adr/` — **VERIFIED** |
| B-P2-2 | P2 | hooks | Scoped pytest selects tests by filename only: core modules commit with their real tests unselected — **VERIFIED** |
| B-P2-3 | P2 | hooks | The scoped-pytest hook never fires for `.sh`, `.ps1`, `.json`, `.yml`, deletions or submodule bumps — **VERIFIED** |
| B-P2-4 | P2 | hooks | 199 of 252 embedded shaders are never validated by any hook — **VERIFIED** |
| B-P2-5 | P2 | hooks | The GLSL validator exits 0 on shaders it cannot see or stage, and on Python syntax errors — **VERIFIED** |
| B-P2-6 | P2 | generators | The spectral-fingerprint generator, its committed output and the live profile tables have fully drifted — **VERIFIED** |
| B-P2-7 | P2 | generators | `gen_fireworks` reads uninitialized memory; the committed scene is not reproducible — **VERIFIED** |
| B-P2-8 | P2 | bench | Every beat-tracker bench driver is dead, and their default outputs overwrite committed results — **VERIFIED** |
| B-P2-9 | P2 | bench | The frozen detector copies are rc.40; the shipped detector is rc.46 — **VERIFIED** |
| P3 | P3 | various | 14 in Part A, 15 in Part B; see the P3 tables |

---

## Part A — packaging, install, launchers

### A-P1-1. `release.sh` overwrites artifacts in the release archive

`tools/packaging/release.sh:107` (default destination
`~/projects/_software-dist/linux`), `189`, `239`, `248`, `280`, `313`.

Re-running for a version already staged (an internal fix with no bump, or a
re-run after a failed rpm step):

- `tar -czf` replaces the old tarball (189);
- `SHA256SUMS` is truncated and rewritten in place, not atomically (239);
- a stale `.asc` is deleted (248);
- the old bundle directory is `rm -rf`'d and its tarball replaced (280, 313).

The Flatpak and Snap manifests pin that tarball's sha256, so a rebuilt
tarball also silently invalidates them.  The owner's standing rule is that
earlier builds are kept and checksums are cumulative.

**Fix:** refuse to write when the artifact exists (or require `--rebuild`
with a suffixed name); build `SHA256SUMS` in a temp file and `mv`; never
`rm -rf` inside the archive.

### A-P1-2. The Windows build deletes the previous zip first

`tools/packaging/build_windows_portable.sh:53-59` (default output
`~/projects/_software-dist`), `285` (`rm -f "$ZIP"`).  Rebuilding the same
version and label removes the earlier bundle before the new one is written; a
failed zip step leaves nothing.

**Fix:** stop if the zip exists, or write to a temp name and rename.

### A-P1-3. The one-line installer runs unverified scripts from a mismatched repo path

`install.sh:14-27` sources `lib.sh` from
`raw.githubusercontent.com/djunicorntears/unicorn-viz/main/tools/install/lib.sh`.
`tools/install/lib.sh:216-222` fetches `fetch_runtime.sh` the same way and
runs it with `bash`.  There is no checksum and no tag or commit.

The path does not match the repository: the branch is `main` while this repo
lands on `master`, and the owner is `djunicorntears` while `origin` here is
`unicorntearsproject/unicorn-viz` and all 43 submodule URLs use `iDoMeteor`.
Whoever controls that path controls every one-line install; if it does not
resolve, every one-line install fails.

**Not confirmed** (no network): who controls that URL and whether it
resolves.  The unverified execution and the three-way owner mismatch are
certain.

**Fix:** publish `lib.sh` and `fetch_runtime.sh` beside `install.sh` on the
release host with their sha256 values embedded in `install.sh`, or inline
them.

### Part A, P2

**A-P2-1. Seat refresh rewinds a drop-in branch — VERIFIED.**
`tools/seat_worktree.sh:128-130` runs `git checkout -q -B "$branch" "$sha"`
whenever the drop-in's `HEAD` is not the pinned commit.  A seat that has
committed in `drop-ins/<x>` but not yet bumped the pointer (the documented
landing order) loses those commits from the branch when anyone re-runs
`tools/seat_worktree.sh <that-seat>`; they survive only in the reflog.  The
comment at 123-125 says work in progress is never reset.  The script can be
pointed at another seat's tree.  A toy repository reproduced the semantics
(2 commits before, 1 after).  **Fix:** act only when `HEAD` is an ancestor of
the pin and the tree is clean; otherwise report and skip.

**A-P2-2. No `GIT_*` scrub.**  `tools/seat_worktree.sh:49-58`, `71-132`.
With an inherited `GIT_DIR` (from a hook), every `git -C` call, including
`remote set-url` (106) and `checkout -B` / `reset --hard` (126-127), targets
that other repository.  This is the leak behind the 2026-10-02 incident.  The
exact damage was not reproduced.  **Fix:** `unset "${!GIT_@}"` at the top, as
`fetch_patched_sources.py` already does.

**A-P2-3. Default install fails on Fedora — VERIFIED.**
`tools/install/lib.sh:121` sources `/etc/os-release` at global scope, which
overwrites the installer's own `VERSION` (for example with
`44 (MATE-Compiz)`).  The channel lookup is skipped and the install stops
with "no source artifact for version 44 (...)".  `--version X` is clobbered
too.  Arch has no `VERSION` line and is unaffected.  CI runs only
`--dry-run --no-deps`, which skips this path.  **Fix:** read os-release in a
subshell and export only `ID` and `ID_LIKE`.

**A-P2-4. Unsigned releases abort the install — VERIFIED.**
`install.sh:148`: the last statement of `resolve_release` is
`[[ -n "$SUMS_ASC_URL" ]] && ...`, called bare at 259 under `set -e`.
`release.sh` without `--sign` (the default) writes no signature, so the
function returns 1.  **Fix:** end the function with `return 0`.

**A-P2-5. Integrity rests on the download host.**  `install.sh:141-142`,
`156-173`, `184-187`, `204-206`.  The tarball sha256 comes from the same
manifest as the URL; a manifest without one only warns; no signature, no
`gpg` or no key only warns; the key itself can be fetched from beside the
manifest; and a signed `SHA256SUMS` that does not list this tarball is
accepted.  **Fix:** embed the release-key fingerprint, require the signed
sums to list the tarball, make a missing sha256 fatal.

**A-P2-6. The 3.11 runtime is unpinned — VERIFIED.**
`tools/packaging/fetch_runtime.sh:55`, `165-179`, `220-234`.  Only the 3.14.7
rows carry a digest in the repo; the `gil` flavor falls back to a checksum
from the same GitHub release as the archive.  `install.sh:248-255` always
bootstraps with that flavor first.  **Fix:** add pins or remove the flavor.

**A-P2-7. Paths that still produce Python below 3.13.**
`unicornviz/remote_objects.py:404` calls `SharedMemory(track=False)`, a 3.13+
argument (core audit P2-12).

| Path | Interpreter |
|---|---|
| `install.sh`, bundled | 3.14.7 free-threaded only on x86_64 with a wheelhouse in the tarball; otherwise stays on 3.11.10 |
| `install.sh --system-python` | whatever `python3` is; no check |
| `tools/install_linux.sh` (clone) | 3.11.10 unless a wheelhouse is present |
| `build_native.sh` rpm/deb | 3.14.7t on x86_64 with a staged wheelhouse; otherwise 3.11.10, no warning |
| `build_windows_portable.sh`, Inno installer | 3.14.7t by default |
| `tools/install_windows.ps1` | system Python 3.11+; installs 3.11 through winget when missing |
| Flatpak 25.08 | 3.13 (GIL build) |
| Snap core24 (never built) | 3.12 |
| `release-installers.yml` | fetches the wheelhouse first, so 3.14.7t |

**Fix:** make free-threaded 3.14 the only default, fail when the wheelhouse
is missing, raise `requires-python` to 3.13.

**A-P2-8.**  `tools/packaging/stage_payload.sh:197-229` checks staged wheels
against `SHA256SUMS` in the wheelhouse folder itself.  A wheel dropped there
with a matching line ships.  `wheelhouse-cp314t.sha256` in the repo is only
consulted by `fetch_wheelhouse.sh`.  **Fix:** check against the committed
file.

**A-P2-9.**  `release.sh:87`, `115-146`, `188`, `193`; `build_native.sh:94`;
`build_windows_portable.sh:94-97`.  `--version` is never compared with the
tree's `__version__` (`--dry-run --version 9.9.9` was accepted).  The payload
is staged separately for source, rpm and deb, minutes apart.  The clean-tree
guard omits `config.dist.toml`, `images/` and every drop-in.  The native and
Windows builds have no guard, and the Windows build never goes through
`release.sh`.  Drop-ins are staged from their working trees, not their pinned
commits.  **Fix:** stage once, assert the staged version, assert `HEAD`
unchanged at the end.

**A-P2-10.**  `build_windows_portable.sh:244-247`, `283-318`.  Nothing
records the zip's hash.  Whatever file is passed as `--vlc-installer` is
copied into `vendor\` and later run on the user's machine.  **Fix:**
cumulative sums file; pinned sha256 for the VLC installer.

**A-P2-11.**  `build_windows_portable.sh:155-167`.  A network error during
the cross-install is reported as "no cp314t build" and the bundle ships
without that dependency (`av`, `hidapi`, torch…), exit 0.  Those requirements
are also unpinned.  **Fix:** an allowlist of dependencies permitted to be
missing.

**A-P2-12.**  `requirements.txt:17-18`;
`tools/gen_third_party_licenses.py:44`, `126`, `190-204`, `228-230`.  The
only importer of `openai` / `anthropic` in the repository is the training
kit's packager; the app does not need them at runtime, yet every install
path installs them.  Their transitive dependencies are absent from
`THIRD_PARTY_LICENSES.md`.  The generator says drop-in dependencies are "not
bundled", but the Windows packs bundle them.  A dependency with no detectable
license renders "UNKNOWN — verify manually" and exits 0.  `--check` currently
fails as out of date (a new `mediapipe` row), and no hook or release script
runs it.  **Fix:** move the two SDKs to an extra; generate the notice from
the staged bundle; fail on UNKNOWN.

**A-P2-13.**  `build_native.sh:65` (`/opt/unicorn-viz`);
`packaging/windows/UnicornViz.iss:34`, `43-44`; `unicornviz/__main__.py:371`.
The wrapper points `UNICORNVIZ_APP_ROOT` at the install prefix and
`_setup_logging` creates `APP_ROOT/logs` with no fallback.  A normal user of
the rpm or deb gets a `PermissionError` at startup.  On Windows the per-user
install is writable; "install for all users" is not, and the shortcut starts
`pythonw`, so the failure is invisible.  The Flatpak has the same shape.
Packages were not built or run.  **Fix:** a per-user state directory.

**A-P2-14.**  `tools/install/lib.sh:179`: `pacman -Sy` without `-u` on one of
the owner's two platforms.  **Fix:** `pacman -S --needed`.

**A-P2-15.**  `tools/install/lib.sh:488-495`, `install.sh:216-219`,
`tools/install/uninstall_linux.sh:28-30`: `rm -rf "$prefix"` with no marker
check and no prompt.  `config.toml`, `runtime/`, `logs/` and recordings live
under the prefix; `--uninstall --prefix "$HOME"` removes the home directory.
**Fix:** require an install marker; remove only what the installer created.

**A-P2-16.**  `tools/install_windows.ps1:210-216`,
`tools/install/dropin_deps.ps1:74`, `tools/install_windows.bat:5-16`.  pip
exit codes are never checked and one helper resets `$LASTEXITCODE` to 0.  In
the `.bat`, `%ERRORLEVEL%` inside a parenthesized block expands at parse
time, and `shift` does not affect `%*`, so `--cli` is passed on and binds to
`-PythonVersion`.  **Fix:** check after each pip call; delayed expansion.

**A-P2-17.**  `tools/launchers/windows/UnicornVizGUI.ps1:131-135`, `151-153`.
`git submodule update --init --recursive --remote` checks out the tip of each
drop-in's branch, leaving the superproject dirty for the next pull.  Only the
root requirements are reinstalled.  Reading stdout to the end and then stderr
on the UI thread deadlocks once stderr fills (SUSPECTED; depends on volume).
All 43 submodule URLs are SSH, so a user without a GitHub key fails here.
**Fix:** drop `--remote`; read both streams asynchronously.

**A-P2-18.**  `UnicornVizGUI.ps1:378`, `386`; `install_windows_gui.ps1:256`,
`264`.  `Start-Process -ArgumentList` joins arguments without quoting, so
under `C:\Users\John Smith\...` the "Installer" and "Open Launcher" buttons
do nothing visible.  **Fix:** embed quotes around path arguments.

**A-P2-19.**  `tools/packaging/fetch_wheelhouse.sh:62`, `103-108` truncates
`SHA256SUMS` in `--dest`, overwrites wheels in place and deletes a wheel on
mismatch.  Pointed at the owner's local wheelhouse it wipes the cumulative
checksum file.  **Fix:** refuse a non-empty destination.

**A-P2-20.**  `build_ft_wheels.sh:55`, `67-70`;
`build_ft_opencv_wheel.sh:69-71`; `ft-opencv/Containerfile:18`, `25`;
`ft-sphn/Containerfile:5`; `build_windows_ft_wheels.py:299-303`.
`manylinux_2_28:latest`, unpinned pip, setuptools, meson, cmake, ninja and
`yum` packages run during the build of wheels that ship.  The `--patched`
path pins the image by digest but installs the same unpinned tools.
`ft-patched/README.md` already lists this under "Not pinned".  **Fix:**
digest-pinned images and a hashed build-requirements file.

**A-P2-21.**  `tools/install/lib.sh:265-272`, `399`: the function echoes
`--find-links <dir>` and the caller word-splits it.  **Fix:** return the
directory and build the array at the call site.

### Part A, P3

| ID | Where | What |
|---|---|---|
| A-P3-1 | `build_native.sh:94`, `246`; `release-installers.yml:144-148` | A dashed version is passed unchanged, bypassing the `~` translation in `release.sh:147` (package ordering effect SUSPECTED) |
| A-P3-2 | `tools/packaging/manifest.py:108` | The channel pointer always moves to the version being written, so rebuilding an older release moves it backwards |
| A-P3-3 | `tools/packaging/audit_sphn_licenses.sh:78-82` | No `pipefail` inside `bash -c`: a failed `cargo tree` yields a "0 crates" notice with exit 0 |
| A-P3-4 | `tools/packaging/demucs_weights.py:69-73` | Weights are checked against an 8-hex-digit prefix only; the cache defaults to a predictable `/var/tmp` path; the files are pickles loaded on user machines |
| A-P3-5 | `build_ft_opencv_wheel.sh:143-152` | A reused source tree is verified by `HEAD` only, so local modifications get built |
| A-P3-6 | `ft-sphn/Containerfile:13` | `rustup-init` from an unversioned URL with a pinned sha256: breaks on the next rustup release |
| A-P3-7 | `tools/install_windows.ps1:54-66` | Assigns `$matches` (the automatic variable); the regex does not match the current `py -0p` format (SUSPECTED) |
| A-P3-8 | `tools/install/windows_deps.ps1:57-59`, `79`, `85` | `$rtmidiOk` is always null, so python-rtmidi is always rebuilt from an unpinned sdist; writes `pydistutils.cfg` into the user profile permanently (SUSPECTED) |
| A-P3-9 | `packaging/windows/UnicornViz.iss:72` | The PATH task writes the wrong key for an all-users install and is not removed on uninstall; no `[UninstallDelete]` |
| A-P3-10 | `tools/install_linux.sh:70` | Run from a seat, it writes through the `.venv` symlink into the live app's environment |
| A-P3-11 | `install.sh:255-276` | A failure after the runtime step leaves a half-populated prefix; the venv is reused without `--clear` across a 3.11 → 3.14t switch |
| A-P3-12 | `tools/seat_worktree.sh:63-65`, `81`, `140`, `150` | Accepts `..` as a seat name (stops on a git error, nothing destroyed); does not check an existing seat's branch; prints the symlink lines even when no link was made |
| A-P3-13 | `tools/packaging/stage_payload.sh:129` | `rm -rf "$DEST"` with only a non-empty check; safe with current callers |
| A-P3-14 | `tools/install/lib.sh:465`; `run.sh:16-20` | Unquoted `Exec=` path; `run.sh` overrides a user-set `SDL_VIDEODRIVER` |

### Part A: clean results

- No secrets, `curl | sh`, `-k`, `--trusted-host` or disabled TLS checks.
- Every in-scope shell script passes `bash -n`.
- The cp314t runtime, the cp314t wheelhouse, every source tarball for the
  wheel builds, the patched forks (40-hex commits) and both patch files are
  pinned and verified against values in the repo before use.  Patch
  provenance is recorded in `ft-patched/README.md` and `recipe.json`; the
  hashes match the files on disk.
- `sudo` is given fixed package lists only.
- No `git clean`, no `find -delete`, no recursive `Remove-Item`.
- `run.sh` forwards arguments and the exit code correctly.

### Part A: dead, duplicated or drifted

- `tools/launchers/linux/unicorn-viz.desktop` hard-codes
  `/home/j/Repos/unicorn-viz`; it works for nobody.
- `tools/launchers/linux/unicorn-viz.sh` duplicates `run.sh` without the
  video-driver selection.
- `tools/install/windows_deps.ps1` and `tools/compat_matrix.sh` target the
  superseded 3.11 / GIL setup.
- `ft-patched/as-built/linux-scripts/*.sh` are archival, with absolute
  `/home/jj/...` paths; the live path is `build_ft_*.sh --patched`.
- The smoke scripts exist twice (`tools/packaging/` and `ft-patched/smoke/`);
  both are used and the difference is deliberate.
- `packaging/snap/snapcraft.yaml` and the Flatpak manifest pin beta.111; core
  is beta.199.
- `README.md:3` says beta.187; core is beta.199 (`version_report.py --plain`
  prints MISMATCH).
- Only the Linux tarball, rpm and deb get manifest entries and checksums.
  The rpm and deb ship no drop-ins; neither Windows pack includes
  `auto-vj-01`.

---

## Part B — hooks, generators, profiling, bench

### B-P1-1. The docs-check hook cannot fail — VERIFIED

`.pre-commit-config.yaml:6`, `tools/check_docs.py:93-94`.  The hook runs
`check_docs.py --warn-only`, which returns 0 on any error, and pre-commit
hides the output of passing hooks.  `check_docs.py --full` exits 1 today with
**40 standing errors**: 35 broken links (mostly `docs/marketing/*` from
`REFERENCE.md` and `docs/README.md`) and 5 missing metadata headers.  A
scratch doc with no metadata and broken links printed four "warnings" and
exited 0.

**Fix:** clear the 40 and drop `--warn-only`; at minimum set `verbose: true`.

### Part B, P2

**B-P2-1. Documentation SOP rules with no code behind them — VERIFIED.**
`tools/check_docs.py:16-32`.  Nothing checks that a doc is linked from an
index.  Metadata is checked only under `docs/planning`, `debug`, `audits`,
`marketing`, `drop-ins/*/docs` and `REFERENCE.md`; `docs/adr/` and
`docs/*.md` are exempt.  **Fix:** an inbound-link check and a wider scope.

**B-P2-2. Scoped pytest is filename matching — VERIFIED.**
`tools/pytest_for_changes.py:44-46`, `81-94`.  Sixteen broad prefixes
(`app.py`, `analyzer.py`, `profiles.py`, `effects/base.py`, `conftest.py`,
`pyproject.toml`…) trigger the full suite.  Otherwise a test is selected only
when its filename shares a token of four or more characters with the changed
file's stem, plus two always-on tests.

| Changed file | Selected | Importing tests missed |
|---|---|---|
| `unicornviz/runtime_state.py` | 1 + 2 | 8 of 9 |
| `unicornviz/__main__.py` | 2 + 2 | 9 of 11 |
| `unicornviz/remote_objects.py` | 1 + 2 | 4 of 5 |
| `unicornviz/safe_store.py`, `ansi/font.py`, `unicornviz/__init__.py` | 2 always-on only | all |
| `drop-ins/auto-vj-01/auto_vj.py` (if passed) | 2 always-on only | all |

Of 60 core files: 18 trigger the full suite, 37 get at least one matched
test, 5 get only the always-on pair.  The pre-push hook still runs the whole
suite, which is why this is P2: the failure surfaces at push, not at commit.
**Fix:** select by import, and fall back to the full suite when nothing
matches.

**B-P2-3. Change types the hook never sees — VERIFIED.**
`.pre-commit-config.yaml:20-23`.  The file filter is `py|toml|txt|yaml`.
`.sh`, `.ps1`, `.json`, `.yml`, deletions and submodule pointer bumps do not
invoke it, although `tests/test_install_lib.py` and
`tests/test_fetch_runtime.py` exist for the shell scripts.  `config.toml`
selects nothing.  **Fix:** widen the filter; treat a gitlink or deletion as a
broad change.

**B-P2-4. Drop-in shaders are never validated — VERIFIED.**
`.pre-commit-config.yaml:28-33`.  Submodule files never appear in the main
repo's staged list, so the `drop-ins/` half of the filter is dead.  None of
the 40 drop-in hook configs runs the validator.  `validate_glsl.py unicornviz`
finds 53 shaders; the full tree has 252.  All 252 compile today.  **Fix:**
add the hook to the drop-in hook template.

**B-P2-5. The validator's blind spots exit 0 — VERIFIED.**
`tools/validate_glsl.py:98-108`, `171-205`.  It uses a real compiler
(`glslangValidator`, compile only).  It extracts only bare string literals
assigned to a name or passed by keyword.  Shaders wrapped in a call, held in
a dict or tuple, built with f-strings or passed positionally are invisible;
names with no recognizable stage are listed as "Skipped"; a Python syntax
error is reported and exits 0.  A scratch file with 9 broken shaders had 7
go unreported.  In the real tree 17 shaders are unvalidated today (10 in
`postfx-01`, 7 in `webcam-01`).  **Fix:** walk every string constant
containing `#version`; fail on unknown stage and on parse errors.

**B-P2-6. Fingerprint generator and live tables have drifted — VERIFIED.**
`tools/gen_spectral_fingerprints.py`, `tools/spectral_fingerprints_out.py`,
`unicornviz/audio/profiles.py:12-20`.  The path from generator output to
`profiles.py` is manual paste.  Of 27 live profiles, 1 matches the tool's
output, 9 differ by up to 0.96 per band, and 17 have `expected_bands` with no
generator output at all.  The `profiles.py` docstring still cites this script
as the provenance of the live numbers.  This is a provenance finding, not a
judgment of the values.  **Fix:** mark both tool files as historical, or add
a test tying the table to its source.

**B-P2-7. `gen_fireworks` — VERIFIED.**  `tools/gen_sim_scenes.py:580-582`.
`np.empty((500, 3))` is filled for 498 particles; the last two are garbage in
every frame.  Thirteen of fourteen regenerated scenes are byte-identical to
the committed ones; `fireworks_burst.usda` differs.  **Fix:** `np.zeros`.

**B-P2-8. Bench drivers are dead — VERIFIED.**
`tools/beat-tracker-bench/bridge_v2.py:51-54`, `batch_311.py:40-43`,
`onset-prototype/run_bench.py:64-67`.  Every input path is hard-coded to one
past session's scratch directory under `/tmp`, which is gone.  Their default
outputs are the committed result files, written in place.  **Fix:** take
`--corpus`; default `--out` outside the repo.

**B-P2-9. Bench detector copies are six releases behind — VERIFIED.**
`onset-prototype/beat_grid_stock.py` is byte-identical to auto-vj-01 commit
`dd03ba3` (2026-09-02, detector rc.40); `beat_grid_e5.py` is that plus a
72-line patch.  The shipped file is rc.46: 3,641 lines added, a duplicated
v3 base class, and the rc.41-46 observation and decision changes.  The
committed bench tables describe rc.40 and nothing says so.  **Fix:** stamp
`results/SUMMARY.md` with the detector version and commit.

### Part B, P3

| ID | Where | What |
|---|---|---|
| B-P3-1 | `tools/validate_glsl.py:56-61` | Stage is inferred by substring: `INVERT_FRAG` contains `VERT` and is compiled as a vertex shader — **VERIFIED** |
| B-P3-2 | `tools/validate_glsl.py:76`, `128`; `tools/check_docs.py:104-109` | Run from another directory, `check_docs.py` checks nothing and exits 0; a nonexistent validator target scans 0 files and exits 0 — **VERIFIED** |
| B-P3-3 | `tools/gen_spectral_fingerprints.py:416`, `469-471`, `497` | Writes "Generated … using claude-opus-4-8" while calling `gpt-4o`; pads a short vector with its last value; cannot reproduce its own committed output; can only write in place — **VERIFIED** |
| B-P3-4 | `tools/gen_sim_scenes.py:18-19`; `tools/generate_ansi_art.py:600-617`; `tools/fetch_acid_ans.py:12`, `44` | Three generators write to cwd-relative paths with no output flag; one writes into the `sims-01` submodule |
| B-P3-5 | `tools/generate_ansi_art.py:403…590` | Color indexes 8-15 are emitted as raw SGR codes the core parser ignores, so that text keeps the previous color — **VERIFIED**.  Output is otherwise deterministic, byte-identical to the committed files, with correct SUB + SAUCE |
| B-P3-6 | `tools/fetch_acid_ans.py:51-73` | HTTPS, hard-coded 18-entry list, no archives, never overwrites.  Weak: unbounded read, no content check, non-atomic write (an interrupted file is "skip (exists)" forever), exit 0 when every download fails |
| B-P3-7 | `onset-prototype/complex_onset.py:196-217` | `_NORM_MIN_HISTORY` is still undefined (09-30 P3-14); the method is dead, so nothing crashes |
| B-P3-8 | `beat-tracker-bench/batch_311.py:59-70` | Decode failures are skipped silently and a partial table is written with exit 0 (306 results under a "311" name) |
| B-P3-9 | `tools/profiling/ft_ab/make_config.py:28-30`, `133` | No argument parser and `main()` runs at import: `-h` creates `./-h/`; `--loaded` overwrites the checkout's mixer state files (it refuses when source and destination are the same) |
| B-P3-10 | `tools/profiling/ft_ab/run_one.sh:31` | The leftover-process check matches the owner's live app helper, so every A/B run aborts while the app is up |
| B-P3-11 | `tools/profiling/projectm_bench.py:193` | Sets `PYSDL2_DLL_PATH` after `import sdl2`; no effect |
| B-P3-12 | `tools/bpm_eval_*.json`, `bpm_eval_report.md` | Orphaned outputs of a training-kit script; nothing in `tools/` reads them |
| B-P3-13 | `tools/pytest_for_changes.py:112`, `70-71` | pytest inherits the hook's `GIT_*` environment (protection is per test today); `./`-prefixed paths miss the broad prefixes (manual use only) |
| B-P3-14 | `tools/audit_pingpong_friends.py` | Finds one invalid reference today (`America 250` names "Grand Finale", a controller, not an effect) but always exits 0 and is wired into no hook or test |
| B-P3-15 | bench adapters; `btrack/btrack_streaming.cpp`; 9 result JSON files | madmom weights are CC BY-NC-SA, BTrack GPL-3.0, Essentia AGPL-3.0.  A GPL-3.0-derivative binding source is committed in an MIT repository (dev-only by its own header).  About 3,000 committed result rows carry `/home/jj/Music/crates/...` paths and so the owner's track library |

### Part B: what works

- The profiling parsers (`perf_frames.py`, `video_decks_report.py`,
  `mixer_store_bench.py`, `ft_ab/summarize.py`) match the log formats the app
  writes today.  None attaches to the live app or edits `config.toml`.
- `--help` works for 13 scripts; it fails only for `btrack/run.py` (unbuilt
  extension).  Ten scripts have no argument parser, so `--help` runs them.
- `complex_onset.ComplexOnsetDetector`, `causal_norm`, `stock_flux_odf` and
  `v3_odf_tracker` still import and are correct.

---

## Rule violations (CLAUDE.md)

- **"Network access is limited to `tools/fetch_acid_ans.py`."**  Also
  networked: `gen_spectral_fingerprints.py` (OpenAI), `madmom/install.sh`,
  `compat_matrix.sh` (git, pip, apt, docker), and every fetch and build
  script in Part A.  The rule text predates the packaging tree.
- **Keep all dist builds; checksums cumulative.**  Not enforced by
  `release.sh`, `build_windows_portable.sh` or `fetch_wheelhouse.sh`.
- **Git History Safety.**  `seat_worktree.sh` can rewind a drop-in branch
  (A-P2-1).
- **Documentation SOP.**  No enforcing gate (B-P1-1, B-P2-1).
- **Versioning.**  `README.md` header is twelve betas behind.
- **Regression Test Discipline.**  No tests exist for the three hook scripts.
- **Security Findings.**  `pip-audit` is a manual-stage hook; nothing runs it
  on commit or push.  (It currently reports 42 Pillow advisories; see the
  core audit.)

---

## What this audit did not do

- No build, download, install, signing or release was run; no Windows host
  and no `pwsh`; no package was installed to test file permissions.
- Skimmed, not read in full: `ft-patched/build_windows_patched_wheels.py`
  (lines 140-890), the form-layout section of `UnicornVizGUI.ps1`
  (176-334), `tools/version_report.py` after line 80, the as-built scripts,
  the four bench adapters, and the middle of `gen_spectral_fingerprints.py`.
- Not read: `ft-patched/smoke/ft_combined_smoke.py`, `gilcheck.py`, the
  `reference/*.json` and `windows-runs/*.json` records, bench result files
  beyond counts.
- Did not confirm who controls the bootstrap URL in A-P1-3.
- Did not fix anything.  A-P2-3 and A-P2-4 are one line each and block the
  default Linux install today.
