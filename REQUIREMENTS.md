# REQUIREMENTS — FloatNote

Authoritative for **what the app must do**. Anything not listed here is out of scope until
moved in via a spec change. IDs (FR/BR/EC/AC/A/D) are stable references used by
`ROADMAP.md` and `AGENTS.md`.

## Confirmed functional requirements

### Window & interaction

- **FR-1** Always-on-top, borderless note window, displayed above other regular applications.
  Scope: macOS and Linux/X11. Wayland: best effort, degradation allowed (see EC-2).
- **FR-2** Drag: pressing and dragging the header bar moves the window anywhere on screen.
  Position is retained for the session and persisted (FR-13).
- **FR-3** Resize: a bottom-right grip resizes the window. Minimum 240×160 px, default
  360×320 px. Size is persisted (FR-13).
- **FR-4** Editing: a multiline plain-text textarea, always editable. Body changes autosave
  debounced 500 ms after the last keystroke, plus a forced flush when the note is hidden,
  the selection changes, or the app quits.
- **FR-7** Minimize: the header minimize button hides the note window and shows the
  **bubble** — a small (≈56×56 px) always-on-top window at the note's last position. The
  bubble is draggable and its position is persisted.
- **FR-8** Restore: clicking the bubble, or pressing the global hotkey, shows the note
  window at its stored geometry and focuses the textarea.
- **FR-9** The window's close button behaves exactly like minimize (FR-7): hides to bubble.
  The application does not exit.
- **FR-10** Quit: only via the bubble's right-click menu item **Quit** (and OS-initiated
  termination such as SIGTERM/Cmd-Q, which must flush pending saves first).

### Notes

- **FR-5** Multiple notes: create and delete notes; switch the visible note with a dropdown
  in the header. Dropdown entries show the derived title (FR-6) and are ordered by most
  recently updated first. Header also has a "＋" (new note) control.
- **FR-6** Title = first non-empty line of the body, whitespace-trimmed, truncated to 40
  characters. Empty body → `"Untitled"`. Titles are derived at render time; there is no
  stored title.
- **FR-12** All notes and settings persist in a single SQLite database; after restart the
  previously selected note is shown.
- **FR-13** Geometry (note window and bubble) is persisted **per display**, keyed by screen
  name. If the saved screen is not connected at startup, geometry is clamped onto a live
  screen.

### Hotkey, lifecycle, instances

- **FR-11** Global hotkey **Cmd+Shift+Space** (macOS) / **Ctrl+Shift+Space** (Linux)
  toggles note-window visibility (hide → bubble, or restore → note). If registration
  fails or the session is Wayland, the app runs normally with the feature disabled and a
  warning logged (EC-2).
- **FR-14** Single instance: launching while already running signals the existing process
  to show the note window, then the new process exits with code 0.
- **FR-15** Startup shows the note window (at persisted geometry) with the bubble visible.
- **FR-16** The application performs **no network activity** ever.

## Business rules

- **BR-1** Deleting the selected note selects the most recently updated remaining note; if
  none remain, a new empty note is created immediately.
- **BR-2** Empty bodies are valid at any time; the note round-trips UTF-8 unchanged.
- **BR-3** Autosave must not cause perceptible typing lag. If a synchronous save ever does,
  the save moves to a background goroutine with last-write-wins semantics.
- **BR-4** A hard kill (crash / `kill -9`) may lose at most the 500 ms debounce window of
  typing (S3).
- **BR-5** The database is opened in WAL mode; exactly one process ever owns it (FR-14).
- **BR-6** Geometry is written on drag/resize **end** and on quit — never per frame.
- **BR-7** All autosaves and flushes happen before process exit on graceful termination.

## User roles & permissions

- Single local user; no accounts, no auth, no roles.
- **macOS OS permissions:** none required (the hotkey uses the Carbon registration API,
  which does not trigger Input Monitoring prompts). The app is unsigned → Gatekeeper
  requires the documented one-time `xattr -dc` workaround (VISION S5).
- **Linux (X11) OS permissions:** none.
- **Wayland:** no prompts; hotkey disabled (FR-11).

## Edge cases

- **EC-1** Display unplugged / changed while saved geometry is off-screen → clamp onto a
  live screen; never spawn invisible or negative-coordinate windows.
- **EC-2** Hotkey unavailable (grab conflict, Wayland) → feature disabled, warning logged,
  bubble/menu remain fully functional; app must not crash or relaunch.
- **EC-3** Stale single-instance lock (owner PID no longer exists) → second launch takes
  over the lock instead of hanging or failing.
- **EC-4** Database missing/corrupt/unreadable → log the error, show one error dialog with
  the path, exit non-zero. **Never** auto-delete or auto-recreate the user's database.
- **EC-5** Very large note (>1 MB) → allowed (SQLite TEXT), no imposed limit; performance
  is not guaranteed beyond that and is a non-goal.
- **EC-6** The bubble must never be hidden — it is the only way back when the hotkey is
  unavailable; quit must not be reachable except through it (FR-10).
- **EC-7** Unicode/RTL/emoji text survives save/load byte-for-byte.

## Acceptance criteria

- **AC-1** (FR-1..3) Note window floats above a normal app, moves by header drag, resizes
  by corner grip, never below 240×160 — verified on macOS and Ubuntu/X11.
- **AC-2** (FR-4, BR-4) Type 10 characters, `kill -9` the app, relaunch → at most the last
  ≤500 ms of characters are missing.
- **AC-3** (FR-5, FR-6) Create 3 notes, type distinct first lines, restart → dropdown shows
  3 correct titles ordered by recency; selecting each shows its body; last-selected note is
  restored.
- **AC-4** (FR-7..9) Minimize → note gone, bubble present at note's last position; click
  bubble → note back at same geometry, textarea focused. Close button does the same.
- **AC-5** (FR-11) Press the hotkey from another application → window toggles within
  200 ms. On Wayland (or with a conflicting grab) → app alive, warning in log, no crash.
- **AC-6** (FR-13, EC-1) Save geometry on display A, unplug A, relaunch → window appears
  clamped inside display B.
- **AC-7** (FR-14, EC-3) Launch twice → one process, note window shown/raised, second exits
  0. Kill first, relaunch → works (stale lock recovered).
- **AC-8** (FR-10, BR-7) Type, immediately Quit from bubble menu → text present after
  relaunch.
- **AC-9** (FR-16) No network connections exist while the app runs.
- **AC-10** (release gate, G4) A macOS `.app` opens after the documented `xattr` command;
  the AppImage runs on a clean Ubuntu LTS.

## Assumptions (not yet user-confirmed behavior — flag on review)

- **A-1** Display name **FloatNote**, binary `floatnote`, module
  `github.com/sambacarlson/floatnote`.
- **A-2** Hotkey combo is a hardcoded constant (FR-11); no key-remapping UI in MVP.
- **A-3** Note counts in the tens; dropdown list is sufficient (no search, no pagination).
- **A-4** English-only UI strings.
- **A-5** Linux MVP is validated on Ubuntu with X11; other distros are best effort.
- **A-6** macOS target = latest two major releases.
- **A-7** Note window has no minimize-to-dock behavior; the dock icon stays visible while
  the app runs.

## Deferred (explicit non-requirements for MVP)

- **D-1** Floating above macOS *full-screen* apps (needs `collectionBehavior` CGO shim).
- **D-2** Wayland global hotkey via XDG shortcuts portal.
- **D-3** Code signing / notarization; Homebrew / AUR distribution.
- **D-4** Markdown, search, tags, themes, tray icon, Windows.
- **D-5** User-configurable hotkey or any settings UI (settings live in SQLite already;
  UI comes later).
