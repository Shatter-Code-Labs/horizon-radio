# horizon-radio Architecture

> This document is the authoritative reference for how horizon-radio is structured.
> All code written for this project must conform to the boundaries and decisions defined here.
> If a decision is not covered here, open an issue before writing code.

**Project:** horizon-radio  
**Organization:** Shatter-Code-Labs  
**License:** GPL-3.0  
**Maintainer:** Shatter-Code-Labs

---

## 1. Project Goal

horizon-radio is a standalone desktop application for Forza Horizon that transforms
the player's own music library into a dynamic, mood-reactive radio station experience.

It is not a playlist switcher. It is not a media remote. It is a radio station that
responds to what is happening on track — adjusting where in a song playback begins,
managing station identity (jingles, station IDs, ads), and restricting interruptions
to natural break points (menus, between races).

The application runs as a standalone window, designed to live on a second monitor
alongside the game. It does not render on top of the game and has no dependency
on game capture or overlay injection.

Secondary goals:
- Serve as a reference-quality open source project under Shatter-Code-Labs
- Be readable and maintainable by a solo contributor indefinitely
- Be approachable enough that others can study and fork it

---

## 2. High-Level Architecture

```
FH6 Data Out (UDP @ 127.0.0.1)
          │
          ▼
┌─────────────────────┐
│   Telemetry Layer   │  Parses raw UDP packets into typed GameState
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│    State Machine    │  Classifies GameState into RadioMood
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Radio Controller  │  Manages seek logic, break queue, cooldowns
└──────────┬──────────┘
           │
      ┌────┴────┐
      │         │
      ▼         ▼
┌──────────┐  ┌──────────────┐
│  Music   │  │ Break Layer  │
│ Backend  │  │  (pygame)    │
└──────────┘  └──────────────┘
      │
      ├── Spotify (Web API)
      └── Local Files (mutagen + pygame)
```

Each layer has one responsibility. No layer may reach across a boundary to call
a layer it does not directly own. Data flows downward. Events flow upward via
callbacks. The UI layer is a read-only consumer of state — it never drives logic.

---

## 3. Layer Definitions

### 3.1 Telemetry Layer

**Location:** `core/telemetry.py`  
**Responsibility:** Listen on a configurable UDP port, parse raw FH6 Data Out
packets into a typed `GameState` dataclass. Nothing else.

**Owns:**
- UDP socket lifecycle
- FH6 packet struct definition and parsing
- `GameState` dataclass

**Does not own:**
- Any game logic or mood classification
- Any awareness of music backends
- Any UI concerns

**Key type produced:**

```python
@dataclass
class GameState:
    speed: float
    rpm: float
    gear: int
    accel: float
    brake: float
    tire_slip_fl: float
    tire_slip_fr: float
    tire_slip_rl: float
    tire_slip_rr: float
    smashable_vel_diff: float
    car_group: int
    race_position: int
    lap_number: int
    current_race_time: float
    is_race_on: bool
    timestamp: float
```

---

### 3.2 State Machine

**Location:** `core/state.py`  
**Responsibility:** Consume a stream of `GameState` updates and emit a `RadioMood`
enum value. Apply hysteresis (debounce) so moods do not flicker on rapid changes.

**Owns:**
- `RadioMood` enum definition
- Mood classification logic
- Hysteresis/debounce thresholds
- Transition rules between moods

**Does not own:**
- Any knowledge of music backends
- Any UDP or network concerns
- Any UI concerns

**Moods:**

```python
class RadioMood(Enum):
    IDLE    = "idle"    # No telemetry / game not running
    MENU    = "menu"    # In menus, between events — break zone
    CRUISE  = "cruise"  # Low speed, low intensity
    SPRINT  = "sprint"  # High speed, sustained acceleration
    DRIFT   = "drift"   # High tire slip angle sustained
    INTENSE = "intense" # High speed + high inputs combined
```

**Seek target ranges per mood:**

| Mood    | Entry point (% of track duration) |
|---------|-----------------------------------|
| CRUISE  | 0 – 25%                           |
| SPRINT  | 40 – 65%                          |
| DRIFT   | 30 – 50%                          |
| INTENSE | 60 – 85%                          |
| MENU    | N/A — break zone, no seek         |
| IDLE    | N/A — playback paused             |

---

### 3.3 Radio Controller

**Location:** `core/radio.py`  
**Responsibility:** The central coordinator. Receives mood changes from the state
machine, instructs the active music backend to seek or play, and manages the break
queue when the game enters a break zone.

**Owns:**
- Seek target calculation (random within mood range)
- Cooldown timer (minimum time between seeks — default 30s)
- Break queue management
- Break cadence rules
- Backend selection at startup (set once from config, not hot-swapped)

**Does not own:**
- Direct UDP/telemetry access
- Audio playback implementation
- UI concerns

**Break cadence rules:**
- Station jingle: every song transition during a break zone
- Station ID clip: every 3rd break zone entry
- Ad clip: every 7th break zone entry
- Ads never stack — one ad per break maximum
- Break clips play via the Break Layer, not the music backend

---

### 3.4 Music Backends

**Location:** `backends/`  
**Responsibility:** Implement the `MusicBackend` abstract interface. Provide
playback control to the Radio Controller without exposing implementation details.

**Abstract interface (`backends/base.py`):**

```python
class MusicBackend(ABC):
    @abstractmethod
    async def play(self) -> None: ...

    @abstractmethod
    async def pause(self) -> None: ...

    @abstractmethod
    async def seek(self, position_ms: int) -> None: ...

    @abstractmethod
    async def get_track_duration_ms(self) -> int: ...

    @abstractmethod
    async def get_current_position_ms(self) -> int: ...

    @abstractmethod
    async def next_track(self) -> None: ...
```

**Implementations:**
- `backends/spotify.py` — Spotify Web API via spotipy, OAuth PKCE, requires Premium
- `backends/local.py` — Local audio files via mutagen (metadata) + pygame (playback)

**Known limitations by backend:**

| Feature             | Spotify          | Local Files |
|---------------------|------------------|-------------|
| Millisecond seek    | ✅               | ✅          |
| Requires Premium    | ✅               | ❌          |
| Official API        | ✅               | N/A         |
| Random entry points | ✅               | ✅          |

**Out of scope for v1:** YouTube Music. No official API exists. May be revisited
in a future release if an official integration path becomes available.

---

### 3.5 Break Layer

**Location:** `core/audio.py`  
**Responsibility:** Manage a secondary pygame audio channel for station break
assets (jingles, station IDs, ads). Independent of the music backend. Ducks
music backend volume during breaks and restores it afterward.

**Owns:**
- pygame mixer initialization and lifecycle
- Break asset loading and playback
- Volume ducking (music to ~20% during breaks, restore after)
- Break asset rotation (avoid back-to-back repeats)

**Does not own:**
- Music backend control
- Break scheduling (that is the Radio Controller's job)

**Asset directory structure:**

```
station_assets/
├── jingles/     # Short stings between songs (3–5s)
├── ids/         # Station ID clips (5–8s)
└── ads/         # Ad clips e.g. Ko-Fi (10–15s max)
```

`station_assets/` is gitignored. Default placeholder assets ship in
`station_assets_default/` and are copied on first run if no assets exist.

---

### 3.6 UI Layer

**Location:** `ui/`  
**Responsibility:** Present configuration and live status to the user via a
NiceGUI native desktop window. The UI is a consumer of state — it reads from
the system and writes to config. It contains no business logic.

**Window behaviour:**
- Launches as a resizable native desktop window
- Default size approximates a music player widget
- Designed to run on a second monitor alongside the game
- Does not render on top of the game
- Scales intelligently — all layout uses proportional sizing

**Pages:**
- `ui/setup.py` — First-run wizard: backend selection, auth flows, port config
- `ui/dashboard.py` — Main view: now playing, station name, playback controls
- `ui/station.py` — Break cadence settings, asset manager
- `ui/playlists.py` — Mood entry point configuration

**Status bar (always visible, bottom of window):**

| Indicator | States |
|-----------|--------|
| Game connection | 🟢 Receiving telemetry / 🟡 Connected, no data / 🔴 Not connected |
| Music backend | 🟢 Connected / 🟡 Authenticating / 🔴 Disconnected |
| Backend label | Displays active backend name (Spotify / Local Files) |

**Developer panel (hidden by default, toggled via status bar):**
- Raw telemetry field values
- Current mood state and transition history
- Seek debug info (last seek position, cooldown timer)

**UI must not:**
- Call telemetry or backend code directly
- Contain mood classification logic
- Block the event loop

---

## 4. Configuration

**Location:** `config.yaml` (gitignored, generated on first run from `config.default.yaml`)

```yaml
telemetry:
  port: 20066

backend: spotify  # spotify | local

spotify:
  client_id: ""
  client_secret: ""
  playlist_uri: ""

local:
  music_dir: ""

radio:
  seek_cooldown_seconds: 30
  break_cadence:
    station_id_every: 3
    ad_every: 7

station:
  name: "horizon-radio"
  assets_dir: "station_assets"
```

`config.default.yaml` is committed to the repo and serves as the canonical
reference for all configuration keys. `config.yaml` is never committed.

---

## 5. Data Flow

```
UDP packet received
  → Telemetry Layer parses → GameState emitted via callback
  → State Machine receives GameState → classifies → RadioMood emitted
  → Radio Controller receives RadioMood
      → if mood is MENU or IDLE:
          → check break cadence rules
          → if break scheduled: Break Layer ducks music, plays asset, restores
      → else:
          → calculate seek target (random % within mood range)
          → if cooldown has elapsed:
              → Music Backend seeks to target position
  → UI Layer receives state snapshot via callback → renders, does not act
```

---

## 6. Dependency Map

```
ui/           →  core/radio.py, config.yaml (read/write)
core/radio    →  backends/base.py, core/audio.py, core/state.py
core/state    →  core/telemetry.py
core/audio    →  pygame
backends/*    →  their respective external libraries only
```

No reverse dependencies. No circular imports. The UI never imports from backends
directly.

---

## 7. Technology Stack

| Concern         | Technology                  | Notes                        |
|-----------------|-----------------------------|------------------------------|
| Language        | Python 3.11+                | Single language throughout   |
| UI framework    | NiceGUI (native mode)       | `ui.run(native=True)`        |
| Spotify API     | spotipy                     | OAuth PKCE, Premium required |
| Local audio     | pygame + mutagen            | Playback + metadata          |
| Break audio     | pygame.mixer                | Secondary channel            |
| Config          | PyYAML + pydantic           | Validation on load           |
| Packaging       | PyInstaller + Inno Setup    | Produces Windows installer   |

---

## 8. Delivery and Packaging

horizon-radio ships as a self-contained Windows installer. Users must not be
required to install Python, pip, or any runtime dependency manually.

**Delivery chain:**

```
Python source
    │
    ▼ PyInstaller
Bundled executable (Python runtime + all dependencies included)
    │
    ▼ Inno Setup
horizon-radio-setup-v{version}.exe
    ├── Checks for WebView2 runtime, installs silently if missing
    ├── Installs application to %PROGRAMFILES%\horizon-radio
    ├── Creates Start Menu shortcut
    ├── Creates optional desktop shortcut
    └── Registers clean uninstaller in Add/Remove Programs
```

**Requirements:**
- Target platform: Windows 10 (up to date) and Windows 11
- WebView2 runtime: handled by installer, not a manual user requirement
- No internet connection required after install (except for Spotify OAuth on first run)

**Release artifacts:**
- One file posted to GitHub Releases: `horizon-radio-setup-v{version}.exe`
- Inno Setup script committed to repo at `installer/horizon-radio.iss`
- PyInstaller spec committed to repo at `horizon-radio.spec`
- GitHub Actions builds the installer automatically on tagged release

---

## 9. What This Architecture Deliberately Excludes

- **No AI-generated music or recommendations** — the user's library, the user's taste
- **No cloud sync or accounts** — config lives locally, always
- **No telemetry sent anywhere** — game data stays on the machine
- **No auto-updates** — users pull releases manually from GitHub
- **No plugin system** — scope is fixed, complexity is the enemy
- **No overlay injection** — runs as a standalone window, not on top of the game

---

## 10. Future Scope (Not In v1)

These are acknowledged but explicitly out of scope until v1 is stable:

- Smart seek using local audio energy analysis (librosa)
- YouTube Music backend (pending official API availability)
- Support for Forza Motorsport
- OBS overlay widget integration
- Per-car-group playlist assignment

---

*Last updated: 2026*  
*Maintainer: Shatter-Code-Labs*
