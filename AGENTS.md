# AGENTS.md — Flask-MDict

A Flask-based MDict dictionary server. Serves dictionary definitions from MDX/MDD files and pre-converted SQLite dictionaries via a web UI.

## Quick start

```bash
pip install -r requirements.txt
mkdir content
# copy .mdx / .mdd / .db dictionary files into content/
python app.py
# open http://127.0.0.1:5248
```

## Commands

| Command | Purpose |
|---|---|
| `python app.py` | Run dev server (default `127.0.0.1:5248`) |
| `python app.py --mdict-dir <path>` | Point to dictionary directory |
| `python app.py --host 0.0.0.0:5248` | Bind all interfaces |
| `python app.py --debug` | Flask debug mode |
| `python app.py --config-file <path>` | Custom JSON config |
| `python app.py --ssl adhoc` | Self-signed HTTPS |
| `run.cmd` | Windows shortcut: `python app.py %*` |

## Architecture

- **Entrypoint**: `app.py` — contains `create_app()` factory and `cli()` argument parser.
- **Blueprint**: `flask_mdict/__init__.py` registers a `'mdict'` Blueprint (url prefix `/mdict`).
- **Routes**: All in `flask_mdict/views.py` (query, resource serving, history, toggle, list).
- **Dictionary backends** (loaded in `helper.init_mdict()`):
  - **MDX/MDD files** → `IndexBuilder2` (`mdict_query2.py`) parses and builds SQLite index caches.
  - **Pre-converted `.db` files** → `DBDict` (`dbdict_query.py`) reads a pre-built schema (tables: `meta`, `mdx`, `mdd`).
- **Translation plugins**: `flask_mdict/plugins/*.py` — each exports `init()` returning a config dict and a `query(content, item)` callable.

## Dictionary loading

- Scans `content/` (or `--mdict-dir`) recursively for `.mdx` and `.db` files.
- `.mdx` files get SQLite index caches generated on first run — written as `.mdx.db` files **alongside** the source `.mdx`. These are build artifacts; include them in `.gitignore`.
- `.db` files must contain `meta` + `mdx` tables (and optionally `mdd`). These are *not* MDX indexes — they are a custom pre-converted format.
- Dictionary enable/disable state is persisted in `flask_mdict.db` (app DB, created in the mdict dir).

## Word Frequency Database (optional)

| File | Source |
|---|---|
| `flask_mdict_wfd.db` | SQLite DB from [ECDICT](https://github.com/skywind3000/ECDICT) |
| `ecdict.sh` | Script to download and import `ecdict.csv` into SQLite |

Without this file, word metadata (tags, frequency, Collins/BNC/COCA scores) and random-word features are unavailable. Place it in `content/` or the app root.

## Plugin system

`flask_mdict/plugins/` — each `.py` file (except `__init__.py`) is auto-loaded.

**Interface** (see `youdao.py` for reference):

```python
def init():
    return {
        'title': 'Plugin Name',
        'uuid': 'unique_id',
        'logo': 'icon.png',
        'query': callable(content, item) -> [html_strings],
        'type': 'app',
        'enable': True,   # whether to auto-enable on load
    }
```

Plugins use the `translators` library. Dependencies: `flask_mdict/plugins/static/` for static assets.

## Build / Deploy

- **PyInstaller**: `py2exe.cmd` builds a single `.exe` via PyInstaller. Copies `flask_mdict_wfd.db` alongside the exe.
- **Config file**: `flask_mdict.json` (auto-generated with defaults if missing). Overrides via `--mdict-dir`, `--host`, `--debug`, `--ssl` CLI flags.
- **App SQLite DB** (`flask_mdict.db`): created in the mdict directory on first run. Tracks history and per-dictionary enable/disable state.

## Gotchas

- **No tests exist** — zero test infrastructure. No linting, type checking, or CI configuration.
- **Python >= 3.6** required.
- `SECRET_KEY` is hardcoded in `app.py` (`"21ffjfdlsafj2ofjaslfjdsaf"`) — not suitable for production.
- Index SQLite DBs (`.mdx.db`) are regenerated when the `.mdx` file modification time changes.
- The `.gitignore` ignores `*.db`, `content/`, `build/`, `dist/`, `.venv/`.
- License: MIT + 996.ICU.
