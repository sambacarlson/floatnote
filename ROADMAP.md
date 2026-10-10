# ROADMAP — FloatNote

Milestones are sequential; dependencies are explicit. **MVP = M0 + M1 + M2** (release
`v0.1.0`). Anything in M3+ is not MVP. Task IDs reference requirements (FR/EC/AC in
`REQUIREMENTS.md`).

## M0 — Foundation (headless, no UI)

Goal: storage and plumbing exist and are proven by tests before any window code.

- [ ] `go mod tidy` scaffold; `cmd/floatnote/main.go` stub that opens/closes the DB and logs
      startup (temporary `main` — real wiring comes in M1).
- [ ] `internal/config`: data-dir resolution (macOS Application Support / XDG data dir),
      paths for db, log, lock, socket; constants (window sizes, debounce, hotkey combo).
- [ ] `internal/store`: open + WAL + `user_version` migration (schema v1 per
      `ARCHITECTURE.md`), `CreateNote/ListNotes/GetNote/UpdateNoteBody/DeleteNote`,
      `GetSetting/SetSetting`. (FR-12, BR-5)
- [ ] `internal/store` unit tests: CRUD round-trip, ordering by `updated_at`, settings
      JSON, UTF-8/emoji round-trip (EC-7), open-on-corrupt-file error path (EC-4).
- [ ] Title derivation + geometry-clamp helpers with unit tests (FR-6, FR-13, EC-1) — pure
      functions, location: `internal/app`.
- [ ] `slog` file handler wired (TECH_STACK logging policy).
- **Depends on:** nothing.
- **Exit gate:** `gofmt -l .` empty, `go vet ./...`, `go test ./...` green.

## M1 — Core UX (the floating app)

Goal: the product exists end-to-end on macOS + X11, data survives restarts.

- [ ] `internal/ui`: borderless note window (FR-1), header bar with dropdown (FR-5),
      ＋ new, minimize (FR-7), close (FR-9), multiline textarea (FR-4).
- [ ] Custom drag widget on header (FR-2) and bottom-right resize grip (FR-3).
- [ ] `internal/ui`: bubble window — small, always-on-top, draggable, click = restore
      (FR-8), right-click menu: Show note / Quit (FR-10).
- [ ] `internal/app.Controller`: selection state, debounced autosave 500 ms + flush points
      (FR-4, BR-3, BR-4), delete-note cascade (BR-1), geometry save on drag-end (BR-6),
      shutdown flush (BR-7).
- [ ] Always-on-top flag on both windows (FR-1).
- [ ] Geometry persistence per display + startup clamp (FR-13, EC-1).
- [ ] Restore selected note on start (FR-12, FR-15).
- [ ] **Spikes (resolve before locking M1 exit):**
  - S-1: borderless Fyne window stays movable/focusable on macOS.
  - S-2: `fyne` screen identity/position API is sufficient for per-display keys (else
    fallback: screen name + bounds heuristic) — record result in `ARCHITECTURE.md`.
  - S-3: Fyne headless `test` package covers header callbacks (informs ui test depth).
- **Depends on:** M0.
- **Exit gate:** automated gates green + manual checklist §M1 below passes on macOS and
  Ubuntu/X11.

## M2 — Platform (ship `v0.1.0`) ← MVP boundary ends here

Goal: the app behaves like an installed desktop app.

- [ ] `internal/hotkey`: register global toggle (FR-11) — mac Carbon, X11 `XGrabKey`;
      runtime Wayland detection → `noop` + warn (EC-2). Spike S-4: evaluate
      `golang.design/x/hotkey` vs hand-rolled CGO; record winner in `TECH_STACK.md`.
- [ ] Single instance: PID lock + unix socket `show` command; stale-lock recovery
      (FR-14, EC-3).
- [ ] SIGTERM/Cmd-Q → graceful shutdown flush (FR-10, BR-7).
- [ ] Error dialog for startup DB failure (EC-4).
- [ ] Packaging: `build/` assets (icon, AppDir), mac `.app` via `fyne package`,
      Linux AppImage via `appimagetool` (FR G4). Build AppImage on the **oldest supported**
      Ubuntu runner (glibc floor).
- [ ] CI: GitHub Actions test matrix (macos/ubuntu: fmt, vet, test, build) + tag-triggered
      release artifacts.
- [ ] README with install instructions incl. unsigned-mac `xattr -dc` workaround (S5).
- [ ] Tag **`v0.1.0`** after full manual checklist.
- **Depends on:** M1.
- **Exit gate:** automated gates + manual checklist §M2 + AC-1..AC-10 signed off.

## M3+ — Later (explicitly not MVP)

| Item                                                            | Req ref    |
| --------------------------------------------------------------- | ---------- |
| macOS full-screen-auxiliary shim (float above fullscreen apps)  | D-1        |
| Wayland hotkey via XDG shortcuts portal                         | D-2        |
| Code signing/notarization, Homebrew tap, AUR                    | D-3        |
| Settings UI (hotkey remap), markdown, search, tray fallback     | D-4, D-5   |
| Log rotation, `golangci-lint` hard-fail, `fyne-cross` if needed | TECH_STACK |

## Manual test checklist (verification gate)

**§M1 — core UX** (macOS + Ubuntu/X11 each):

1. Launch → note + bubble visible, note on top of other apps (FR-1, AC-1).
2. Header drag moves window; corner grip resizes; both persist across restart (AC-1, AC-6).
3. Type → wait 1 s → restart → text present (AC-3).
4. Type → `kill -9` immediately → relaunch → ≤500 ms of text lost (AC-2, S3).
5. 3 notes: create, switch via dropdown, titles = first lines, order by recency, restart
   restores last note (AC-3).
6. Minimize → bubble at note's position; click bubble → note back, textarea focused (AC-4).
7. Close button behaves as minimize (AC-4); quit only via bubble menu (FR-10).
8. Delete selected note → next-most-recent selected; delete last → fresh note (BR-1).
9. Unplug/ignore display change → relaunch clamps onto live screen (AC-6).

**§M2 — platform:**

10. Hotkey toggles from another app < 200 ms (AC-5); with grab pre-taken → warn + app alive
    (EC-2).
11. Second launch pokes first and exits 0; after `kill -9`, relaunch recovers lock (AC-7).
12. Quit immediately after typing → text persisted (AC-8).
13. mac `.app` opens after `xattr -dc`; AppImage runs on clean Ubuntu LTS (AC-10, S5).
14. No network connections while running (AC-9).

## Versioning plan

| Event                                | Tag                                       |
| ------------------------------------ | ----------------------------------------- |
| First code lands (end M1)            | `v0.1.0-alpha` (optional, local tag only) |
| MVP complete (M2 exit)               | **`v0.1.0`**                              |
| Post-MVP features (M3 items landing) | `v0.2.0`, `v0.3.0`, …                     |
| Fixes only                           | `v0.1.x` patch bumps                      |
| Stable API/spec freeze               | `v1.0.0`                                  |
