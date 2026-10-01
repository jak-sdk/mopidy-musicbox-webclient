# Reviving mopidy-musicbox-webclient

Notes from getting this extension running again on modern Python / Mopidy (2026).

## What was broken

Installing the last PyPI release (`Mopidy-MusicBox-Webclient` 3.1.0) on Python 3.12+ fails at extension load:

```text
ModuleNotFoundError: No module named 'pkg_resources'
```

Cause: `__init__.py` used `pkg_resources.get_distribution(...)` for the package version.
`pkg_resources` comes from setuptools, which is no longer shipped in venvs by default,
and recent setuptools (82+) removed `pkg_resources` entirely.

## What we changed

1. **Version lookup** — switched to `importlib.metadata.version(...)` in
   `mopidy_musicbox_webclient/__init__.py` (same approach as current Mopidy).
2. **Runtime deps** — dropped the `setuptools` runtime requirement (no longer
   needed once `pkg_resources` is gone).
3. **Packaging** — moved metadata, deps, and the `mopidy.ext` entry point into
   `pyproject.toml`; removed `setup.py` / `setup.cfg`. Dev tools use uv
   dependency groups (`test`, `lint`, …). Requires Python >= 3.11.

There is still **no separate frontend build** for normal use: the JS/CSS under
`mopidy_musicbox_webclient/static/` is served as-is.

## Build a wheel (usual workflow)

```bash
uv build --wheel
# -> dist/mopidy_musicbox_webclient-3.1.0-py3-none-any.whl
```

Copy the `.whl` to the Mopidy host, then:

```bash
uv pip uninstall --python ~/mopidy/.venv/bin/python Mopidy-MusicBox-Webclient
uv pip install --python ~/mopidy/.venv/bin/python ./mopidy_musicbox_webclient-3.1.0-py3-none-any.whl
```

Restart Mopidy and open:

```text
http://<host>:6680/musicbox_webclient/
```

Editable install into an existing Mopidy venv:

```bash
uv pip install --python ~/mopidy/.venv/bin/python -e .
```

## Local Python tests (optional)

```bash
uv sync --group test
uv run pytest
```

If sync tries to compile `pygobject`/`pycairo` from PyPI, create the venv with
system site packages so distro PyGObject (`gi`) can satisfy Mopidy:

```bash
uv venv --system-site-packages
uv sync --group test
```

**Known gap:** against Mopidy 4, most tests still fail because `mopidy.config.Proxy`
was removed. Extension load / serving still works; tests need updating for Mopidy 4’s
config API.

## Longer-term modernisation (not done yet)

- Align further with Mopidy’s extension template (`src/` layout, setuptools-scm,
  ruff/pyright): https://docs.mopidy.com/stable/guides/extensiondev/
- Fix tests for Mopidy 4
- Refresh or drop the legacy JS toolchain (`package.json` / karma / phantomjs / tox node envs)

## Alternatives (if you only need a UI)

- **Iris** (`Mopidy-Iris`) — actively maintained, richest web UI
- **Mopidy-Mobile** — lightweight mobile remote
- **Mowecl** — React-based dual-panel client
