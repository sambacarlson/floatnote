# TECH_STACK — FloatNote

## Languages & runtime

| Item | Choice | Notes |
|---|---|---|
| Language | Go (modern, ES-equivalent = Go 1.22+ idioms) | Primary (and only) language |
| Minimum version | **Go 1.27.2** (`go.mod` says `go 1.27.2`) | Matches installed toolchain |
| Module path | `github.com/sambacarlson/floatnote` | Must match the GitHub repo |
| CGO | Required | Forced by Fyne (OpenGL) and the hotkey shims |

## Core dependencies

| Dependency | Purpose | Constraint |
|---|---|---|
| `fyne.io/fyne/v2` | GUI: windows, widgets, always-on-top, screens | Latest stable **v2.x** pinned in `go.mod` at first implementation; floor 2.5. No third-party theme code |
| `modernc.org/sqlite` | SQLite driver, pure Go (`database/sql`) | Chosen over `mattn/go-sqlite3` to keep the storage layer C-free; swap is one line if ever needed |
| stdlib `log/slog` | Structured logging to file | No third-party logger |
| stdlib `time`, `sync` | Debounce, locking | — |

Hotkey implementation: prefer `golang.design/x/hotkey` **if** a spike proves it registers
without prompts on macOS and grabs cleanly on X11; otherwise hand-rolled CGO shims
(`RegisterEventHotKey` / `XGrabKey`). Decide during M2 spike; record outcome here.

**Rule: no other dependencies without updating this file and getting approval**
(see `AGENTS.md`). Prefer stdlib.

## Platform build requirements

- **macOS:** Xcode Command Line Tools (CGO + OpenGL).
- **Linux (build):** `gcc`, `pkg-config`, `libgl1-mesa-dev`, `xorg-dev`
  (`libxcursor-dev libxrandr-dev libxinerama-dev libxi-dev libxxf86vm-dev`).
- **Linux (run, AppImage target):** X11 session; Wayland runs degraded (EC-2).
- **Wayland build:** no special requirement — Wayland differences are runtime-detected
  (`XDG_SESSION_TYPE`), hotkey falls back to `noop`.

## Coding conventions

- Standard Go layout: `cmd/floatnote/main.go` + `internal/...` only. No `pkg/`, no
  generics-for-their-own-sake.
- `gofmt` is law; `go vet` must pass. Optional `golangci-lint` in CI, default config.
- Errors: returned, never panicked in non-`main` code; wrap with `fmt.Errorf("...: %w", err)`
  adding operation context (`update note %d: %w`). Sentinel errors only for branch-worthy
  cases (e.g. `store.ErrNotFound`).
- No `fmt.Print`/`println` in library code — `slog` only (`Info` for lifecycle, `Warn` for
  degraded features, `Error` for failures).
- Comments only where the *why* is non-obvious; no comment noise (repo-wide rule).
- UI strings: English literals in `internal/ui` (A-4).
- Concurrency: touch `store` only through `Controller`; widget state changes on Fyne's
  main goroutine or via Fyne's locked APIs.

## Error handling & logging policy

- Startup failures (DB open/migrate, lock) → `slog` + single error dialog → exit ≠ 0 (EC-4).
- Degraded features (hotkey, Wayland) → `slog.Warn`, continue (EC-2).
- Save failures → `slog.Error`, retry once on next flush; never block typing (BR-3).
- Log: append-mode file `<dataDir>/floatnote.log`, default `slog` text handler, level Info.
  No rotation (non-goal for MVP).

## Testing

| Layer | Tool | Policy |
|---|---|---|
| `store`, `app` logic | stdlib `testing` + temp-file/in-memory SQLite | Unit tests required for all CRUD, settings, debounce-flush, title derivation, geometry clamp logic |
| `ui` | Fyne `test` package (headless) | Smoke tests only: widgets build, title derivation renders, callbacks fire |
| Windowing / hotkey / drag | **Manual checklist** in `ROADMAP.md` | Cannot be automated meaningfully in MVP — checklist is part of the verification gate |
| Commands | `go test ./...`, `go vet ./...`, `gofmt -l .` (must print nothing), `go build ./...` | Run before any task is "done" |

## Versioning & releases

- **Semantic versioning** starting at `v0.1.0` (MVP release = end of M2).
- Pre-1.0: breaking/spec changes bump **minor** (`v0.2.0`), fixes bump **patch**
  (`v0.1.1`); after 1.0: standard major/minor/patch.
- Git tags `vX.Y.Z` are the only source of released versions; artifacts are built by CI
  from tags. Binary self-reports `dev` unless stamped via
  `-ldflags "-X main.version=..."` at release build (future task, not MVP-blocking).
- Commits use semantic prefixes: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`,
  `build:`, `ci:`, `chore:`.

## CI / packaging tooling

- **CI:** GitHub Actions — `test` job (macos-latest + ubuntu-latest: fmt/vet/test/build),
  `release` job on tags (build `.app`, AppImage, attach to Release).
- **Packaging:** `fyne package -os darwin` for `.app`; `appimagetool` for AppImage
  (AppDir under `build/`). `fyne-cross` only as fallback if CI builds prove flaky.
