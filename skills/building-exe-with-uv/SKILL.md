---
name: building-exe-with-uv
description: Use when setting up a Python project with uv, building Windows .exe executables with PyInstaller, migrating from pip/requirements.txt to uv, or sharing Python project environments with team members
---

# Building Python Executables with uv

## Overview

Full lifecycle of a uv-based Python project: initialization → dependency management → development → .exe build → project sharing. uv unifies Python installation, virtual environments, and package management into a single tool.

## When to Use

- Starting a new Python project
- Migrating from pip/requirements.txt to uv
- Building a Python project into a .exe
- Sharing a consistent dev environment with team members

## 1. Project Initialization

```bash
# Basic app project (flat layout)
uv init my-app
# Result: my-app/ (main.py, pyproject.toml, .python-version, README.md)

# Package project (src layout - recommended for larger projects)
uv init --package my-app
# Result: my-app/src/my_app/__init__.py + pyproject.toml

# Specify Python version
uv init --python 3.12 my-app
```

**Decision guide:**

| Scenario | Choice |
|----------|--------|
| Single script, simple tool | `uv init` (flat) |
| Multiple modules, team project, planned distribution | `uv init --package` (src layout) |

## 2. Dependency Management

```bash
# Runtime dependencies
uv add requests click rich

# Build/dev tools must be added as dev dependencies
uv add --dev pyinstaller pytest ruff

# Custom groups
uv add --group test pytest pytest-cov

# Migrate from requirements.txt
uv add -r requirements.txt
```

**Resulting pyproject.toml:**

```toml
[project]
name = "my-app"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "requests>=2.31.0",
    "click>=8.1.7",
]

[dependency-groups]
dev = [
    "pyinstaller>=6.0",
    "pytest>=8.0",
    "ruff>=0.4.0",
]
```

> **Always add PyInstaller with `--dev`.** It's a build tool, not needed at runtime.

## 3. Environment Sync and Execution

```bash
uv sync              # Install all dependencies (auto-creates .venv)
uv sync --no-dev     # Production only
uv sync --frozen     # Install from lockfile only (for CI)

uv run main.py       # Run in project environment
uv run pytest        # Run tests
```

`uv run` automatically verifies environment consistency before execution. No manual venv activation needed.

## 4. .python-version File

Create a `.python-version` file at the project root to pin the Python version:

```bash
uv python pin 3.12    # Create/update .python-version file
```

```
3.12
```

When this file exists, `uv sync` will automatically download and install the specified Python version if it's not already available. Always commit this file to Git.

## 5. Test Environment Setup

### pyproject.toml pytest Configuration

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
```

- `testpaths`: Specifies the test directory — just `uv run pytest` is enough to run all tests
- `addopts`: Default options (verbose + short traceback)

### Test Directory Structure

```
my-app/
├── src/my_app/
│   ├── __init__.py
│   ├── core.py
│   └── utils.py
├── tests/
│   ├── conftest.py          # Shared fixtures
│   ├── test_core.py         # Maps to src/my_app/core.py
│   └── test_utils.py        # Maps to src/my_app/utils.py
├── pyproject.toml
├── .python-version
└── uv.lock
```

**Naming convention:** `tests/test_{module}.py` — 1:1 mapping with source modules.

### conftest.py: Global UTF-8 Encoding Setup

The Windows default encoding (cp949) causes `UnicodeDecodeError` during file I/O. Set it globally in `conftest.py`:

```python
"""Shared test fixtures and configuration."""

import builtins

_original_open = builtins.open


def _utf8_open(*args, **kwargs):
    """Default to UTF-8 encoding for text mode file operations."""
    if "b" not in (args[1] if len(args) > 1 else kwargs.get("mode", "r")):
        kwargs.setdefault("encoding", "utf-8")
    return _original_open(*args, **kwargs)


builtins.open = _utf8_open
```

> **Alternative:** Explicitly specifying `encoding='utf-8'` on every `open()` call in source code is safer. The conftest patch should only be used within the test environment.

### Running Tests

```bash
uv add --dev pytest              # Add pytest
uv run pytest                    # Run all (testpaths auto-applied)
uv run pytest tests/test_core.py # Single file
uv run pytest -k "test_parse"    # Name matching
uv run pytest --co               # Collect only (list tests without running)
```

## 6. .exe Build (PyInstaller)

### Basic Build

```bash
# Single-file CLI app
uv run pyinstaller --onefile main.py

# Single-file GUI app (no console window)
uv run pyinstaller --onefile --windowed --name MyApp main.py

# With icon
uv run pyinstaller --onefile --windowed --icon=assets/icon.ico --name MyApp main.py
```

### Including Data Files (Windows uses `;` separator)

```bash
uv run pyinstaller --onefile --add-data "src/myapp/data;data" main.py
```

### resource_path Helper (required for --onefile)

With `--onefile`, PyInstaller extracts to a temp directory at runtime. Always use this helper to access bundled resources:

```python
import sys
import os

def resource_path(relative_path: str) -> str:
    """Return resource path for both PyInstaller bundle and dev environment."""
    try:
        base_path = sys._MEIPASS  # PyInstaller temp path
    except AttributeError:
        base_path = os.path.abspath(".")
    return os.path.join(base_path, relative_path)

# Usage example
config_path = resource_path("data/config.json")
```

### Spec File Workflow (complex builds)

```bash
# 1. Generate spec file
uv run pyi-makespec --onefile --windowed --name MyApp main.py

# 2. Edit MyApp.spec (hidden imports, data files, excluded modules, etc.)
# 3. Build from spec
uv run pyinstaller MyApp.spec
```

**Key spec file settings:**

```python
a = Analysis(
    ['main.py'],
    datas=[('src/myapp/data', 'data')],
    hiddenimports=['pkg_resources.py2_warn'],
    excludes=['matplotlib', 'numpy.tests', 'pytest'],  # Exclude unnecessary modules → smaller size
)

exe = EXE(
    pyz, a.scripts, a.binaries, a.zipfiles, a.datas,
    name='MyApp',
    console=False,       # GUI app
    icon='assets/icon.ico',
    upx=True,
    upx_exclude=['vcruntime140.dll', 'python3.dll'],
)
```

### Resolving Hidden Imports

If `ModuleNotFoundError` occurs when running the .exe:

```bash
# Method 1: CLI flag
uv run pyinstaller --onefile --hidden-import=requests.packages.urllib3 main.py

# Method 2: Add to hiddenimports list in spec file

# Method 3: Auto-collect (inside spec file)
from PyInstaller.utils.hooks import collect_submodules
hiddenimports = collect_submodules('requests')
```

## 7. Nuitka Alternative (optional)

| Aspect | PyInstaller | Nuitka |
|--------|------------|--------|
| Build speed | Fast | Slow (C compilation) |
| Runtime speed | Same as Python | Faster |
| File size | Large | Can be smaller |
| Setup difficulty | Low | High (C compiler required) |

```bash
uv add --dev nuitka
uv run python -m nuitka --mode=onefile --windows-disable-console main.py
```

> Nuitka requires Visual Studio Build Tools or MinGW on Windows. Test with `--mode=standalone` first, then switch to `--mode=onefile`.

## 8. Project Sharing

### Files to Include in Version Control

| File | Commit | Reason |
|------|--------|--------|
| `pyproject.toml` | Yes | Project configuration |
| `uv.lock` | Yes | Exact dependency version reproducibility |
| `.python-version` | Yes | Pin Python version |
| `*.spec` (if customized) | Yes | Build configuration |
| `.venv/` | No | Auto-generated by uv |
| `dist/`, `build/` | No | Build artifacts |

### .gitignore

```gitignore
.venv/
dist/
build/
__pycache__/
*.pyc
*.egg-info/
```

### Team Onboarding

```bash
# After cloning, just run this
uv sync
# uv automatically:
# 1. Reads .python-version and downloads/installs that Python if missing
# 2. Creates .venv/
# 3. Installs exact versions from uv.lock

uv run main.py  # Ready to run immediately
```

> **Always commit `.python-version`.** This file ensures team members get the same Python version automatically with just `uv sync`.

### CI/CD (GitHub Actions)

```yaml
- uses: astral-sh/setup-uv@v4
- run: uv sync --frozen
- run: uv run pytest
- run: uv run pyinstaller --onefile --name MyApp main.py
- uses: actions/upload-artifact@v4
  with:
    name: MyApp
    path: dist/MyApp.exe
```

## Quick Reference

| Task | Command |
|------|---------|
| Create project | `uv init my-app` |
| Package project | `uv init --package my-app` |
| Add dependency | `uv add requests` |
| Add dev tool | `uv add --dev pyinstaller` |
| Migrate requirements.txt | `uv add -r requirements.txt` |
| Sync environment | `uv sync` |
| Run script | `uv run main.py` |
| Build .exe (CLI) | `uv run pyinstaller --onefile main.py` |
| Build .exe (GUI) | `uv run pyinstaller --onefile --windowed main.py` |
| Generate spec file | `uv run pyi-makespec --onefile main.py` |
| Build from spec | `uv run pyinstaller MyApp.spec` |
| Pin Python version | `uv python pin 3.12` |
| Add test framework | `uv add --dev pytest` |
| Run tests | `uv run pytest` |
| List tests | `uv run pytest --co` |

## Common Mistakes

| Mistake | Correct Approach |
|---------|-----------------|
| Adding PyInstaller as runtime dep | `uv add --dev pyinstaller` |
| Not committing `uv.lock` | Always commit (key to reproducibility) |
| Manually editing `uv.lock` | Never do this — uv manages it |
| Hardcoding resource paths with `--onefile` | Use `resource_path()` helper |
| Using `:` in `--add-data` on Windows | Use `;` (`"src;dest"`) |
| `pip install uv` | Use the standalone install script |
| Pinning `==` versions in `pyproject.toml` | Use `>=` ranges — lockfile handles pinning |
| Committing `.venv/` | Add to `.gitignore` |
| Ignoring hidden import errors | Use `--hidden-import` or fix spec file |
| Jumping to Nuitka `--onefile` directly | Test with `--standalone` first |
| Omitting `encoding` in `open()` on Windows | Always specify `encoding='utf-8'` |
| Not setting `testpaths` for tests | Add `testpaths = ["tests"]` to `pyproject.toml` |
