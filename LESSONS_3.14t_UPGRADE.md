# Lessons: attempted upgrade to free-threaded Python 3.14 (3.14t)

**Date:** 2026-09-05
**Outcome:** Attempted, then **fully reverted to 3.13**. Free-threading is blocked by two
upstream packages. Plain (GIL-enabled) 3.14 would have worked.

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
2. **Plain 3.14 is available whenever wanted** and is a clean upgrade — orjson, numpy,
   pillow and opencv all ship working 3.14 wheels. Steps: `uv python pin 3.14`, bump
   `requires-python` to `>=3.14`, `rm -rf .venv && uv sync`.
3. **Retry free-threading when both gaps close.** Nothing in this codebase needs to change;
   these are purely upstream wheel-availability gaps. Check with:

   ```bash
   uv pip download --python-version 3.14t opencv-python orjson
   ```

   or inspect PyPI filenames for a `cp314t` tag.
4. **Free-threading may not be worth much here anyway.** It pays off only for CPU-bound work
   spread across threads; nothing in these scripts is threaded. Single-threaded code is
   typically somewhat *slower* on the free-threaded build.
