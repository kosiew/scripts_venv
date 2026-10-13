# Lessons: evaluating an upgrade to free-threaded Python 3.15 (3.15t)

**Date:** 2026-10-13
**Outcome:** **Deferred.** Not attempted — wheel availability checked first, and two
dependencies lack `cp315t` wheels. Retry in 1–3 months.

---

## Context

- Current environment: CPython 3.14.6 free-threaded (`.python-version` = `3.14+freethreaded`),
  migrated on 2026-10-09. See `LESSONS_3.14t_UPGRADE.md`.
- CPython 3.15.0 final was released 2026-10-09.
- Installed uv is 0.11.21 (June 2026), which only knows `cpython-3.15.0b2` — **upgrade uv
  first** (`brew upgrade uv` / `uv self update`) before any 3.15 attempt.

---

## Wheel audit (PyPI, latest releases, 2026-10-13)

| Package | `cp315` | `cp315t` | Notes |
|---|---|---|---|
| numpy, pillow, matplotlib, pandas, lxml, contourpy, kiwisolver, charset-normalizer | ✅ | ✅ | Need lock bumps to latest versions |
| **curl-cffi** (via `yfinance`) | ❌ | ❌ | Only `cp314-cp314t` and `cp310-abi3`. `abi3` is unusable on free-threaded builds, so uv would source-build against libcurl-impersonate — expect failure (same pattern as orjson in the 3.14t attempt). |
| **pyyaml** | ❌ | ❌ | No 3.15 wheels at all. Sdist usually builds, but may lose the libyaml C extension. |

---

## Why defer

1. **curl-cffi is a hard blocker** for 3.15t (yfinance depends on it).
2. 3.15.0 was days old; ecosystem wheels lag a release by weeks to months.
3. The 3.14t migration had only just landed; stacking another interpreter change adds churn.
4. Little upside: nothing in these scripts is threaded or CPU-bound (see 3.14t notes).

Plain (GIL) 3.15 is closer to viable — curl-cffi's `abi3` wheel works there, leaving only
pyyaml as a source build — but would give up free-threading.

---

## Retry checklist

```bash
uv self update        # or: brew upgrade uv — must list a 3.15.0 final
uv python list | grep 3.15
uv pip download --python-version 3.15t --no-deps -d "$(mktemp -d)" curl-cffi pyyaml
# Both must resolve to .whl files, not .tar.gz
```

When they do, follow the 3.14 procedure:

```bash
uv python install 3.15t
uv python pin 3.15t
# edit pyproject.toml: requires-python = ">=3.15"
rm -rf .venv
uv sync
.venv/bin/python -c "import sys; print(sys.version, sys._is_gil_enabled())"
uv run pytest
```

Remember the 3.14 gotchas: install the exact managed build before pinning, and cross-check
`uv pip list` against `pyproject.toml` for ad-hoc packages before `rm -rf .venv`.
