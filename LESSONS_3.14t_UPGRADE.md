# Lessons: attempted upgrade to free-threaded Python 3.14 (3.14t)

**Date:** 2026-09-05
**Outcome:** Free-threading attempted, then **reverted** — it is blocked by two upstream
packages. The environment was subsequently upgraded to **plain (GIL-enabled) 3.14.6**,
which works fine; see "Follow-up" at the end.

---

## Context

`/Users/kosiew/GitHub/scripts_venv/` is a **uv project**, not a bare venv:

- `pyproject.toml` + `uv.lock` declare the environment; the venv lives at `.venv/`
- `~/scripts_venv` is a **symlink** to `~/GitHub/scripts_venv`
- Scripts in other repos hard-code the interpreter via shebang, e.g.
  `#!/Users/kosiew/scripts_venv/.venv/bin/python` (Raycast commands)

So the environment is changed by re-pinning and re-syncing, never by editing `.venv/` by hand.

Starting state: CPython 3.13.14, `.python-version` = `3.13`, `requires-python = ">=3.13"`.
Tooling: uv 0.11.21, macOS aarch64 (Apple Silicon).

---

## Steps that were run

```bash
cd /Users/kosiew/GitHub/scripts_venv
git checkout -b 3.14t

uv python install 3.14t              # downloads cpython-3.14.6+freethreaded (~25 MB)
uv python pin 3.14t                  # writes ".python-version" = "3.14+freethreaded"

# edit pyproject.toml: requires-python = ">=3.14"; remove the opencv-python line

rm -rf .venv
uv sync
```

Note: `uv python pin 3.14t` normalises the file contents to `3.14+freethreaded`.
The `t` suffix is what selects the free-threaded build.

### Verifying you are actually free-threaded

```bash
.venv/bin/python -VV
# Python 3.14.6 free-threading build (main, Jun 11 2026, 03:55:38) [Clang 22.1.3]

.venv/bin/python -c "import sys; print(sys._is_gil_enabled())"
# False
```

---

## What worked

- The venv built and all 14 declared dependencies imported cleanly with the GIL off.
- `numpy` and `pillow` — the two with compiled extensions — publish real `cp314t` wheels.
- No package fell back to a source build once `opencv-python` was removed.
- The shebang kept working: `.venv/bin/python` was re-pointed at `python3.14t`
  automatically, so dependent Raycast/Alfred scripts needed no edits.

---

## Blocker 1 — `opencv-python` (known before starting)

`opencv-python` ships only `cp37-abi3` wheels plus an sdist:

```
opencv_python-5.0.0.93-cp37-abi3-macosx_13_0_arm64.whl
opencv_python-5.0.0.93.tar.gz
```

The **stable ABI (`abi3`) is not usable on free-threaded builds** — free-threading needs
wheels tagged `cp314t` specifically. With no usable wheel, uv falls back to compiling
OpenCV from the sdist via CMake. That build ran past a 9-minute timeout without
finishing and commonly fails outright on macOS.

**Only consumer:** the "pencilize clipboard" pencil-sketch tool, in two copies —
`raycast/raycast-scripts/python-commands/pencilize_clipboard.py` and
`alfred-workflows/workflows/xizun.pencilize_clipboard/pencil.py`. Usage is shallow:
`imread → bilateralFilter → cvtColor → medianBlur → adaptiveThreshold → bitwise_and → imwrite`.
Most of that ports to Pillow + numpy; `bilateralFilter` (edge-preserving blur) has no
direct Pillow equivalent, so a port would look somewhat different.

## Blocker 2 — `dspy` → `orjson` (the one that killed it)

`dspy` is pure Python (`py3-none-any`), but it depends on **`orjson`, which has no
`cp314t` wheel in any release** up to and including 3.12.0. Building from source fails —
the Rust/maturin build errors out:

```
error: command ['maturin', 'pep517', 'build-wheel', ...] returned non-zero exit status 1
hint: `orjson` was included because `dspy` depends on `orjson`
```

`dspy` is load-bearing: `alias_git_cli.py:21` imports it and calls `dspy.configure` at
line 60, and `tests/conftest.py` pulls that in. Without it, **all 44 test files fail at
collection**.

On plain 3.14 this is a non-issue — orjson publishes proper `cp314-cp314` arm64 wheels.

---

## The confusing failure mode: `dspy` shadowed by a data directory

Removing the venv produced this, 44 times:

```
AttributeError: module 'dspy' has no attribute 'configure'
```

Note it is **`AttributeError`, not `ModuleNotFoundError`**. Two causes combined:

1. **`dspy` was installed ad-hoc and is not in `pyproject.toml`.** `rm -rf .venv` destroyed
   it and `uv sync` had no reason to reinstall it.
2. **`python-scripts/dspy/` is a data directory** (holding `commit-message.json` and
   `commit-message-path.txt`, both tracked in git) with no `__init__.py`. Under PEP 420 it
   is a *namespace package portion*. While the real `dspy` was in site-packages, the real
   regular package won the import; once it was gone, only the empty namespace portion
   remained — so the import succeeded but the module had no attributes.

**Two lessons here:**

- Any package installed ad-hoc into this venv is invisible to `pyproject.toml` and will be
  silently pruned by the next `rm -rf .venv` + `uv sync`. Add real dependencies properly:
  `uv add dspy`.
- A compatibility audit based on `pyproject.toml` alone is incomplete when packages have
  been installed ad-hoc. Cross-check with `uv pip list` against the declared set before
  rebuilding an environment.

---

## Revert procedure (what was done)

```bash
cd /Users/kosiew/GitHub/scripts_venv
git restore pyproject.toml uv.lock .python-version
rm -rf .venv
uv sync
uv pip install dspy        # not tracked in pyproject.toml — see above
```

Verified afterwards: Python 3.13.14, `cv2` 4.13.0, `dspy` 3.3.1 with `configure` present,
and the full suite green — **462 passed**.

---

## Recommendations

1. **Track `dspy`.** Run `uv add dspy` on `main`. Until then this failure recurs on every
   environment rebuild, disguised as an `AttributeError`.
2. **Plain 3.14 — done.** Completed on branch `3.14`; orjson, numpy, pillow and opencv all
   ship working 3.14 wheels. See "Follow-up" below for the steps and one gotcha.
3. **Retry free-threading when both gaps close.** Nothing in this codebase needs to change;
   these are purely upstream wheel-availability gaps. Check with:

   ```bash
   uv pip download --python-version 3.14t opencv-python orjson
   ```

   or inspect PyPI filenames for a `cp314t` tag.
4. **Free-threading may not be worth much here anyway.** It pays off only for CPU-bound work
   spread across threads; nothing in these scripts is threaded. Single-threaded code is
   typically somewhat *slower* on the free-threaded build.

---

## Follow-up: the plain 3.14 upgrade (completed)

Done on branch `3.14`. All 462 tests pass on CPython 3.14.6 with the GIL enabled.
`cv2` 4.13.0, `dspy` 3.3.1 and `orjson` 3.12.0 — the three packages that blocked
free-threading — all work here without incident.

```bash
cd /Users/kosiew/GitHub/scripts_venv
uv python install 3.14               # <- do not skip; see the gotcha below
uv python pin 3.14
# edit pyproject.toml: requires-python = ">=3.14"
rm -rf .venv
uv sync
```

### Gotcha: `uv python pin 3.14` can still select the free-threaded build

The first attempt produced a **free-threaded** venv despite pinning plain `3.14`, and the
sync then failed with:

```
error: Distribution `opencv-python==4.13.0.92` can't be installed because it doesn't have
a source distribution or wheel for the current platform
```

Cause: uv prefers **managed** interpreters over system ones (Homebrew's `python3.14`), and
at that moment the only *managed* 3.14 installed was `cpython-3.14.6+freethreaded`, left
over from the free-threading attempt. A bare `3.14` request matched it.

Fix: `uv python install 3.14` to fetch the plain managed build, then re-sync.

**Always confirm the interpreter after pinning — do not trust the pin alone:**

```bash
.venv/bin/python -VV
.venv/bin/python -c "import sys; print(sys._is_gil_enabled())"   # True on plain 3.14
```

A plain build prints `Python 3.14.6 (main, ...)`; a free-threaded one prints
`Python 3.14.6 free-threading build (main, ...)`.

### Check for ad-hoc packages before destroying the venv

The `dspy` incident above was caused by `rm -rf .venv` pruning an untracked package.
Snapshot first, and diff afterwards, so nothing disappears silently:

```bash
uv pip freeze | sed 's/==.*//' | sort > /tmp/before.txt
# ... rm -rf .venv && uv sync ...
uv pip freeze | sed 's/==.*//' | sort > /tmp/after.txt
comm -23 /tmp/before.txt /tmp/after.txt      # anything listed was lost
```

This was run for the 3.14 upgrade and came back empty.

### The lockfile shrinks a lot — this is expected

`uv.lock` lost ~520 lines on this upgrade. That is not packages disappearing: raising
`requires-python` to `>=3.14` prunes resolution branches that existed only for older
interpreters. The package count was unchanged at 251, and `dspy`, `opencv-python`,
`orjson`, `numpy` and `pillow` were all still present.

---

## Follow-up: free-threaded 3.14t migration (completed 2026-10-09)

Both blockers were removed in this repo rather than waiting on upstream:

- `opencv-python` — the Raycast pencilize script was ported to Pillow + numpy
  (output matches OpenCV to within 0.02% of pixels), then the dependency was removed.
  The Alfred copy still imports `cv2`, but its workflow calls `/usr/local/bin/python3`,
  which no longer exists, so it was already dead.
- `orjson` — gone with `dspy`, which was removed as a dependency. `orjson` 3.13.0 still has
  no `cp314t` wheel.

Steps: snapshot `uv pip freeze`, `uv add click`, `uv python pin 3.14t`, `rm -rf .venv`,
`uv sync`. Result: `Python 3.14.6 free-threading build`, `sys._is_gil_enabled()` → `False`,
package snapshot identical before/after, 689 tests pass.

### Checking wheels before migrating

`uv pip compile --python-version 3.14t` is rejected by uv 0.11.21 (it cannot parse the `t`).
Point `--python` at a real free-threaded interpreter instead, and resolve from the lockfile
so you test the versions you actually run:

```bash
uv venv -q -p 3.14t /tmp/ft
uv export --frozen --all-groups --no-hashes --no-emit-project -o /tmp/locked.txt
uv pip sync --python /tmp/ft/bin/python --only-binary :all: /tmp/locked.txt
```

Resolving `pyproject.toml` fresh instead picked newer versions, among them `typer` 0.27.3,
which no longer depends on `click`. That exposed tests importing `click` without declaring
it, so `click` is now a direct dependency.

### Remaining caveat: `lxml` re-enables the GIL

`lxml` 6.1.3 ships `cp314t` wheels but does not declare GIL-free safety, so importing it
re-enables the GIL for the rest of the process (with a `RuntimeWarning`). It arrives via
`yfinance`, and `bs4` uses it automatically when installed. Scripts that import `bs4` or
`yfinance` therefore run with the GIL on. Everything else stays GIL-free.

`PYTHON_GIL=0` forces it off but is inherited by subprocesses: GIL-enabled interpreters
(e.g. the `llm` CLI on 3.13) abort with `Python runtime state: preinitialized`, which broke
two tests. Do not set it globally.

Check a module with:

```bash
.venv/bin/python -c "import sys, lxml.etree; print(sys._is_gil_enabled())"
```

### Reverting

```bash
uv python pin 3.14      # plain managed 3.14 is already installed
rm -rf .venv && uv sync
```

---

## Follow-up: re-applying 3.14t via Homebrew (2026-10-09)

The pins above said free-threaded, but the venv on disk did not match:

```
.python-version      3.14+freethreaded
pyproject.toml       requires-python = ">=3.14"
.venv/pyvenv.cfg     version_info = 3.13.7   <-- home = /opt/homebrew/opt/python@3.13/bin
```

The migration commit (`5dde040`) touched only `.python-version`, `pyproject.toml`,
`uv.lock` and this file — `.venv/` itself was never rebuilt, so it stayed on 3.13.7.

**Lesson: the pins are not the environment.** `.python-version` is a *request*; only
`.venv/pyvenv.cfg` and `python -VV` tell you what is installed. Check the venv, not the pin
— this is the same warning as the plain-3.14 gotcha above, in the opposite direction.

### Blocker: the local `uv` is too old to fetch a stable 3.14t

```bash
uv --version        # uv 0.7.12 (b3d7f7977 2025-06-11)  -- at ~/.cargo/bin/uv
uv self update      # error: uv was installed through an external package manager
```

`uv` here is a `cargo install` build from June 2025, so `self update` refuses and its
bundled python-build-standalone manifest predates stable 3.14t. The only free-threaded
build it offers is a beta:

```
cpython-3.14.0b2+freethreaded-macos-aarch64-none    <download available>
```

So `uv python install 3.14t` — the route used earlier in this document — is not available
on this machine. It is also the only `uv` on the box (nothing in `/opt/homebrew/bin` or
`~/.local/bin`); the `uv 0.11.21` referenced earlier is gone.

### Route taken: Homebrew's `python-freethreading`

Homebrew ships a stable free-threaded build as a separate formula, bottled:

```bash
brew info python-freethreading     # stable 3.14.8 (bottled)
brew install python-freethreading  # installs as /opt/homebrew/bin/python3.14t
```

The old `uv` still *discovers* it even though it cannot download one itself, so the
existing pin keeps working unchanged:

```bash
uv python find '3.14+freethreaded'
# /opt/homebrew/opt/python-freethreading/bin/python3.14t

uv sync --python '3.14+freethreaded'
```

Note the formula is `python-freethreading`, not `python@3.14t`, and it is independent of
`python@3.14` — both can be installed side by side.

### Result

```
Python 3.14.8 free-threading build (main, Sep 30 2026, 17:55:09)
sysconfig Py_GIL_DISABLED = 1
sys._is_gil_enabled()     = False
.venv/pyvenv.cfg home     = /opt/homebrew/opt/python-freethreading/bin
```

225 packages, every one from a wheel — no source builds. The before/after `uv pip freeze`
diff (the `comm -23` check above) came back empty: nothing lost. `pyproject.toml`,
`uv.lock` and `.python-version` all already matched, so the tree stayed clean — this was
purely a venv rebuild.

Tests: **688 passed, 1 failed** in `~/GitHub/python-scripts`. The failure is unrelated to
the interpreter; see below.

The `lxml` caveat above was reconfirmed on 3.14.8 — `lxml` 6.1.3 still ships a real
`cp314t` wheel without declaring GIL-free safety. Bisecting every declared dependency, the
only two that re-enable the GIL are `bs4` and `yfinance`, both via `lxml.etree`.
`PYTHON_GIL=0` was again left unset.

### Stale path in this document

The header above says `~/scripts_venv` is a symlink to `~/GitHub/scripts_venv`. That is no
longer true: `~/GitHub/scripts_venv` does not exist, and `~/scripts_venv` is now the real
git checkout of `git@github.com:kosiew/scripts_venv.git`. Shebangs pointing at
`/Users/kosiew/scripts_venv/.venv/bin/python` are unaffected.

### The one failing test: the `llm` CLI venv

`tests/test_alias_llm.py::test_llm_fragments_loaders_real_cli` shells out to
`/Users/kosiew/GitHub/llm/.venv/bin/llm`. Two separate problems, neither caused by 3.14t:

**1. That venv's interpreter was a dangling symlink** (fixed). It was built on Python 3.10:

```
.venv/bin/python3.10 -> /usr/local/opt/python@3.10/bin/python3.10   # gone
```

Homebrew's `python@3.10` has since been removed, so exec'ing the `llm` script failed with
`FileNotFoundError` on the *script* path — a misleading errno: the file existed, its
interpreter did not. **A `FileNotFoundError` on a script that plainly exists means a broken
shebang.** Rebuilt on 3.13:

```bash
cd ~/GitHub/llm
rm -rf .venv && uv venv -p 3.13 .venv
uv pip install --python .venv/bin/python -e .
uv pip install --python .venv/bin/python 'openai>=1.55.3,<2'   # see below
```

`llm --version`, `models list`, `aliases list`, `keys list`, `templates list` and
`logs list` all work again.

**2. Unpinned `openai` drifted across a major version.** `setup.py` asks for
`openai>=1.55.3`, which now resolves to **openai 3.26.1** — and openai 3.x swapped its
`httpx` dependency for **`httpx2`**. `llm/models.py` does a plain `import httpx`, which was
only ever satisfied transitively, so the fresh install broke immediately:

```
ModuleNotFoundError: No module named 'httpx'
```

Constraining to `openai<2` restores `httpx` 0.28.1 and matches the 1.97.0 the old venv had.
**Lesson: a transitively-satisfied import is a latent break.** It survives only as long as
some dependency keeps pulling the package in; `llm` should depend on `httpx` directly.

**3. The test cannot pass against this checkout** (not fixed — needs a decision).
`~/GitHub/llm` is pinned at an upstream commit from 2025-01-19, version **0.19.1**, which
has no `fragments` command at all; `llm fragments loaders` is parsed as a prompt and exits
2. The test asserts the three loaders (`github:`, `issue:`, `pr:`) printed by the
**`llm-fragments-github`** plugin, and `llm fragments` itself only arrived in llm 0.24.

The old venv cannot have passed this test either — its `llm.egg-link` pointed at this same
0.19.1 checkout with no plugins installed — so the "689 passed" recorded earlier was from a
machine or checkout that is no longer reproducible here.

Making it pass requires updating the checkout and adding the plugin, e.g.:

```bash
cd ~/GitHub/llm
git remote add upstream https://github.com/simonw/llm.git
git fetch upstream && git merge upstream/main      # 0.19.1 -> 0.24+
uv pip install --python .venv/bin/python -e . llm-fragments-github
```

`origin` is the fork `kosiew/llm` and is level with local `main`, so this is a real fork
update, not a fast-forward of someone else's work. Left undone pending that call.
