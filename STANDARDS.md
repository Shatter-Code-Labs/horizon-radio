# horizon-radio Development Standards

> This document defines the coding standards, conventions, and quality rules for
> horizon-radio. All code in this repository must conform to these standards.
> These rules exist to keep the codebase readable, maintainable, and consistent
> by a solo contributor over a long period of time.
>
> If a rule is not covered here, default to PEP 8 and use good judgment.
> If you disagree with a rule, open an issue before deviating from it.

**Project:** horizon-radio  
**Organization:** Shatter-Code-Labs  
**Maintainer:** Shatter-Code-Labs

---

## 1. Language and Runtime

- Python 3.14 or higher. No compatibility with earlier versions is required or maintained.
- Python 3.14 was chosen specifically because its EOL (October 2030) covers the full active
  lifecycle of Forza Horizon 6. This decision is documented in STANDARDS.md and should not
  be changed without a corresponding rationale.
- All code is typed. Type annotations are required on every function signature —
  parameters and return types. No exceptions.
- `from __future__ import annotations` is included at the top of every module.

---

## 2. Project Structure

```
horizon-radio/
├── main.py                    # Entry point only — no logic
├── horizon-radio.spec         # PyInstaller spec
├── config.default.yaml        # Committed default config
├── config.yaml                # Gitignored user config (generated on first run)
├── core/
│   ├── __init__.py
│   ├── telemetry.py           # UDP listener + FH6 packet parser
│   ├── state.py               # Mood state machine
│   ├── radio.py               # Radio controller
│   └── audio.py               # Break layer (pygame)
├── backends/
│   ├── __init__.py
│   ├── base.py                # MusicBackend abstract class
│   ├── spotify.py             # Spotify backend
│   └── local.py               # Local files backend
├── ui/
│   ├── __init__.py
│   ├── setup.py               # First-run wizard
│   ├── dashboard.py           # Main view
│   ├── station.py             # Station settings
│   └── playlists.py           # Mood entry point config
├── installer/
│   └── horizon-radio.iss      # Inno Setup script
├── station_assets_default/    # Default placeholder assets (committed)
│   ├── jingles/
│   ├── ids/
│   └── ads/
├── station_assets/            # User assets (gitignored)
├── tests/
│   ├── __init__.py
│   ├── test_telemetry.py
│   ├── test_state.py
│   └── test_radio.py
├── ARCHITECTURE.md
├── STANDARDS.md
├── CONTRIBUTING.md
├── README.md
├── LICENSE
├── requirements.txt
├── .gitignore
└── .github/
    └── workflows/
        └── release.yml        # GitHub Actions release build
```

**Rules:**
- `main.py` contains only the entry point. It initialises configuration, wires
  dependencies, and starts the application. No business logic lives here.
- Every directory that contains Python files has an `__init__.py`.
- Test files mirror the module they test: `core/state.py` → `tests/test_state.py`.

---

## 3. Naming Conventions

### Files and Directories
- All lowercase, words separated by underscores: `station_assets/`, `test_telemetry.py`
- No hyphens in Python file or directory names (hyphens only in the repo name and
  installer artifacts)

### Python Identifiers
| Kind | Convention | Example |
|------|------------|---------|
| Module | `snake_case` | `telemetry.py` |
| Class | `PascalCase` | `RadioController` |
| Function / method | `snake_case` | `calculate_seek_target()` |
| Variable | `snake_case` | `current_mood` |
| Constant | `UPPER_SNAKE_CASE` | `DEFAULT_SEEK_COOLDOWN` |
| Private attribute | `_leading_underscore` | `_break_counter` |
| Enum | `PascalCase` class, `UPPER_SNAKE_CASE` members | `RadioMood.CRUISE` |
| Type alias | `PascalCase` | `GameStateCallback` |

### Names Must Be Descriptive
- No single-letter variables outside of loop indices and list comprehensions.
- No abbreviations unless they are universally understood in context
  (`rpm`, `url`, `ms` for milliseconds are acceptable; `rctrl`, `gst` are not).
- Boolean variables and functions that return booleans are prefixed with
  `is_`, `has_`, or `can_`: `is_race_on`, `has_assets`, `can_seek`.

---

## 4. File Structure

Every Python file follows this order, with a blank line between each section:

```python
# 1. Module docstring
"""
Brief description of what this module owns and is responsible for.
"""

# 2. Future annotations import (always first)
from __future__ import annotations

# 3. Standard library imports
import asyncio
import time
from dataclasses import dataclass
from enum import Enum

# 4. Third-party imports
import pygame
import spotipy

# 5. Local imports
from core.state import RadioMood
from backends.base import MusicBackend

# 6. Constants
DEFAULT_SEEK_COOLDOWN: int = 30

# 7. Type aliases (if any)
MoodCallback = Callable[[RadioMood], None]

# 8. Classes and functions
```

**Rules:**
- Imports are always absolute. No relative imports (`from .state import` is not allowed).
- Imports are never done inside functions or methods unless there is an explicit,
  documented reason (e.g. avoiding a circular import that cannot be resolved otherwise).
- `import *` is never used.

---

## 5. Docstrings and Comments

### Module Docstrings
Every module has a docstring at the top that states in plain English what this
module owns and is responsible for. One short paragraph. No bullet lists.

```python
"""
Manages the mood state machine for horizon-radio. Consumes GameState updates
from the telemetry layer and emits RadioMood values via callback. Applies
hysteresis to prevent mood flickering on rapid input changes.
"""
```

### Class Docstrings
Every class has a docstring describing its purpose and any important invariants.

```python
class RadioController:
    """
    Central coordinator for horizon-radio playback logic.

    Receives RadioMood changes, instructs the active music backend to seek
    to the appropriate position, and schedules station break assets via the
    break layer. Does not own telemetry or UI concerns.
    """
```

### Function Docstrings
Functions with non-obvious behaviour or parameters have docstrings. Simple
getters and setters do not require them. Use plain English, not parameter lists.

```python
def calculate_seek_target(self, mood: RadioMood, duration_ms: int) -> int:
    """
    Returns a random seek position in milliseconds within the entry point
    range defined for the given mood. The range is a percentage of the
    total track duration.
    """
```

### Inline Comments
- Comments explain *why*, not *what*. If the code needs a comment to explain
  what it does, it should be rewritten to be clearer instead.
- Comments are written in full sentences with correct capitalisation.
- No commented-out code is committed. Use git history instead.

```python
# Good — explains why
# Debounce mood transitions to prevent seek spam on rapid input changes.
if time.monotonic() - self._last_transition < MOOD_DEBOUNCE_SECONDS:
    return

# Bad — explains what (the code already does this)
# Check if time since last transition is less than debounce threshold
if time.monotonic() - self._last_transition < MOOD_DEBOUNCE_SECONDS:
    return
```

---

## 6. Functions and Methods

- One function does one thing. If a function needs a comment to separate it into
  sections, it should be split into smaller functions.
- Maximum function length: 40 lines. If a function exceeds this, split it.
  This is a strong guideline, not an absolute rule — document the reason if exceeded.
- Functions that can fail must either raise a typed exception or return an explicit
  result type. Silent failures are not acceptable.
- Async functions are used for all I/O operations (network, file, socket).
  Blocking calls are never made from async contexts without wrapping in an executor.

---

## 7. Error Handling

- Every exception that is caught must be handled or re-raised with context.
  Bare `except: pass` is never acceptable.
- External failures (network, file not found, device unavailable) are caught at
  the boundary and converted into application-level status updates that the UI
  can display. They do not propagate as unhandled exceptions to the user.
- Custom exception types live in `core/exceptions.py` and are used for
  application-level errors.

```python
# Good
try:
    await self._backend.seek(target_ms)
except SpotifyException as exc:
    logger.error("Spotify seek failed: %s", exc)
    self._status.set_backend(ConnectionStatus.ERROR)

# Bad
try:
    await self._backend.seek(target_ms)
except Exception:
    pass
```

---

## 8. Configuration

- All configuration is loaded once at startup via `config.yaml` and validated
  by a pydantic model defined in `core/config.py`.
- No module reads `config.yaml` directly. Configuration is passed as a typed
  object at construction time (dependency injection).
- No hardcoded values that belong in config. If a value is something a user
  might reasonably want to change, it lives in `config.yaml`.
- Constants that are not user-configurable (e.g. packet struct format, FH6 field
  offsets) are defined as module-level constants in the relevant module.

---

## 9. Logging

- All logging uses the standard library `logging` module. No `print()` statements
  in production code. `print()` is only acceptable in one-off debug scripts.
- Log levels are used correctly:
  - `DEBUG` — detailed diagnostic information, off by default
  - `INFO` — normal application events (startup, connection, mood changes)
  - `WARNING` — unexpected but recoverable situations
  - `ERROR` — failures that affect functionality
  - `CRITICAL` — failures that require the application to stop
- Logger instances are created at module level:
  ```python
  import logging
  logger = logging.getLogger(__name__)
  ```
- Log messages are lowercase, written as plain sentences, and never include
  f-strings — use `%s` formatting so the string is only built if the level is active:
  ```python
  logger.info("mood transition: %s → %s", previous_mood, new_mood)
  ```

---

## 10. Testing

- Tests live in `tests/` and mirror the module structure of the project.
- The test runner is `pytest`.
- Every public function in `core/` has at least one test.
- Tests are named descriptively:
  `test_mood_transitions_to_drift_on_sustained_slip` not `test_mood_1`.
- Tests do not depend on external services. Spotify and local file I/O are mocked.
- Tests do not depend on each other. Every test is fully independent.

---

## 11. Copyright Headers

Every Python file begins with a copyright header, after the module docstring:

```python
"""
Module docstring here.
"""
# Copyright (C) 2026 Shatter-Code-Labs
# SPDX-License-Identifier: GPL-3.0-or-later
```

---

## 12. Git Conventions

### Commit Messages
Commit messages follow this format:

```
<type>(<scope>): <short description>

<optional body — explain why, not what>
```

**Types:**
| Type | When to use |
|------|-------------|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `test` | Adding or updating tests |
| `chore` | Build process, dependencies, tooling |
| `style` | Formatting only, no logic change |

**Scopes** match the module or layer: `telemetry`, `state`, `radio`, `audio`,
`spotify`, `local`, `ui`, `installer`, `config`.

**Examples:**
```
feat(state): add INTENSE mood for high-speed high-input classification
fix(spotify): handle token refresh failure without crashing
docs(architecture): add delivery and packaging section
chore(installer): add WebView2 bootstrapper to Inno Setup script
```

**Rules:**
- Subject line is 72 characters maximum.
- Subject line does not end with a period.
- Subject line is written in the imperative mood: "add", not "added" or "adds".
- Body is wrapped at 72 characters.
- No emoji in commit messages.

### Branching
- `main` is always releasable. No broken code is committed to `main`.
- Feature work is done on branches named `feature/<scope>-<description>`:
  `feature/state-drift-detection`
- Bug fixes are done on branches named `fix/<scope>-<description>`:
  `fix/spotify-token-refresh`
- Branches are deleted after merging.

### Tagging
- Releases are tagged with semantic versioning: `v1.0.0`, `v1.1.0`, `v1.0.1`
- Tags are annotated: `git tag -a v1.0.0 -m "Release v1.0.0"`
- A tag on `main` triggers the GitHub Actions release build automatically.

---

## 13. Dependencies

- All dependencies are pinned to exact versions in `requirements.txt`.
- Dependencies are not added casually. Every new dependency must solve a problem
  that cannot be reasonably solved with the standard library or existing dependencies.
- No dependency is added to solve a problem that is only needed once or in tests.

---

## 14. What These Standards Deliberately Prohibit

- No `print()` in production code
- No bare `except: pass`
- No relative imports
- No `import *`
- No commented-out code committed to the repo
- No hardcoded user-configurable values
- No business logic in `main.py` or the UI layer
- No blocking I/O in async contexts
- No single-letter variable names outside loops

---

*Last updated: 2026*  
*Maintainer: Shatter-Code-Labs*
