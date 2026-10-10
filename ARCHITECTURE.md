# ARCHITECTURE — FloatNote

One OS process, one SQLite file, two windows. No servers, no plugins, no IPC with anything
but itself.

## Component overview

```
                     ┌────────────────────────────────────────────┐
                     │ cmd/floatnote (main)                       │
                     │  wiring, startup order, shutdown flush     │
                     └──────┬──────────────┬──────────────┬───────┘
                            │              │              │
              ┌─────────────▼──┐   ┌───────▼────────┐   ┌─▼──────────────┐
              │ internal/ui    │   │ internal/app   │   │ internal/hotkey│
              │ NoteWindow     │◄──┤ Controller     │◄──┤ Register()     │
              │ BubbleWindow   │   │ selection      │   │ darwin|x11|noop│
              │ Drag/Resize    │   │ debounce saves │   └────────────────┘
              │ widgets        │   │ quit/flush     │
              └────────────────┘   │ instance lock  │
                                   │ + unix socket  │
                                   └───────┬────────┘
                              ┌────────────▼─────────────┐
                              │ internal/store           │
                              │ SQLite (modernc)         │
                              │ notes + settings, v1 DDL │
                              └────────────┬─────────────┘
                              ┌────────────▼─────────────┐
                              │ internal/config          │
                              │ paths, constants, build  │
                              └──────────────────────────┘
```

Dependency rule: `ui → app → store → config`; `hotkey` and `config` depend on nothing but
the stdlib. `store` never imports Fyne — the storage layer must be testable headless.

## Responsibilities

| Package | Responsibility | Explicitly NOT responsible for |
|---|---|---|
| `cmd/floatnote` | Compose everything in order: config → store → controller → windows → hotkey; handle signals; final flush | Business logic, SQL, widget code |
| `internal/config` | Data-dir resolution (mac/Linux), file paths (db, log, lock, socket), all constants (sizes, debounce, hotkey combo) | Reading settings values (that's `store`) |
| `internal/store` | Open/migrate DB (WAL, `PRAGMA user_version`), CRUD for `notes`, typed get/set for `settings`; plain Go values, `error` returns | Caching, UI state, debouncing |
| `internal/app` | `Controller`: selected note, note list ordering, debounced autosave (`time.AfterFunc`), geometry save on drag-end, quit/flush orchestration, single-instance lock + unix-socket server | Drawing anything; SQL strings (delegates to `store`) |
| `internal/ui` | Fyne windows/widgets: header (dropdown, new, minimize, close), textarea, drag & resize handles, bubble, quit context menu; renders `Controller` state | Persisting anything directly — goes through `Controller` |
| `internal/hotkey` | `Register(combo, fn) error` behind build tags: `darwin` (CGO Carbon `RegisterEventHotKey`), `linux` X11 (`XGrabKey`), fallback `noop` returning a "disabled" error | Deciding what a toggle means (calls `Controller`) |

## Data flows

**Edit → disk**
`textarea.OnChanged → Controller.SetBody(id, text) → debounce 500 ms → store.UpdateNoteBody`

**Switch note**
`dropdown changed → Controller.SelectNote(id) → flush pending save → ui renders new body`

**Minimize / close**
`header button → Controller.HideNote() → store geometry → ui hides NoteWindow, shows Bubble`

**Hotkey / second instance**
`hotkey fn | socket "show" (from another process) → Controller.Toggle() or Show() →
ui show/hide + focus textarea` — these arrive on non-UI goroutines and must go through
`Controller` (mutex-guarded), never touch `store` directly.

**Quit**
`bubble menu Quit → Controller.Shutdown(): flush debounce → store.Close → lock/socket
cleanup → os.Exit(0)`; SIGTERM/Cmd-Q route to the same path.

## Persistence

One SQLite file: `<dataDir>/floatnote.db`.

```sql
PRAGMA journal_mode = WAL;
CREATE TABLE notes (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  body       TEXT NOT NULL DEFAULT '',
  created_at INTEGER NOT NULL,            -- unix ms
  updated_at INTEGER NOT NULL             -- unix ms
);
CREATE INDEX idx_notes_updated_at ON notes(updated_at DESC);
CREATE TABLE settings (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- schema version via PRAGMA user_version (=1 for MVP)
```

Settings keys: `note.geometry.v1`, `bubble.geometry.v1` (JSON: `{"screen","x","y","w","h"}`,
`w/h` omitted for bubble), `last_note_id`. Key names are versioned (`v1`) so future formats
can migrate by key.

`<dataDir>` = `~/Library/Application Support/FloatNote/` on macOS;
`${XDG_DATA_HOME:-~/.local/share}/floatnote/` on Linux. Log at `<dataDir>/floatnote.log`
(`slog` file handler, append, no rotation in MVP — non-goal). Lock file
`<dataDir>/floatnote.lock` (PID); unix socket in the per-user runtime dir
(`$XDG_RUNTIME_DIR`, fallback `$TMPDIR`/`/tmp` with UID suffix).

## Process & concurrency model

- Single process. Fyne owns the main goroutine; all widget work happens on it or via
  Fyne's internally-locked APIs (safe from goroutines).
- `Controller` state (selected id, pending body, visible state) guarded by one `sync.Mutex`.
- Debounce = `time.AfterFunc(500ms, save)`; re-armed on every keystroke; `Shutdown` and
  `SelectNote` drain any pending timer before proceeding (BR-7).
- The socket listener is one goroutine reading short line commands (`show`); it only calls
  into `Controller`.
- SQLite access from the debounce timer goroutine and the UI goroutine is serialized by the
  `Controller` mutex — `store` itself also serializes via a single `*sql.DB` pool of 1
  connection for writes.
- UI updates triggered from the hotkey/socket goroutines go through `Controller` callbacks
  that the `ui` package registered at wiring time.

## System boundary

Talks only to: the window manager (position, always-on-top), the filesystem (db, log, lock,
socket), and itself over the unix socket. **No network, no keychain, no clipboard daemon,
no tray, no other processes.**

## Deployment topology

- Build artifacts: single Go binary; mac `.app` wrapper via `fyne package` (Info.plist
  only, no helper processes); Linux AppImage (one mounted dir, one binary).
- Distribution: GitHub Releases (semver-tagged), unsigned for MVP.
- Runtime footprint: 1 process, 1 DB, 1 log file per user. No daemons, no services.
