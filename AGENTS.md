# AGENTS.md — Working rules for AI coding agents

Repo: `github.com/sambacarlson/floatnote` (Go, spec-first). Read this before touching
anything.

## Authority: which document wins

| Order | Document | Answers |
|---|---|---|
| 1 | `VISION.md` | Why the project exists; non-goals are binding |
| 2 | `REQUIREMENTS.md` | **What** must work — FR/BR/EC/AC IDs are the contract |
| 3 | `ARCHITECTURE.md` + `TECH_STACK.md` | **How** — components, deps, conventions |
| 4 | `ROADMAP.md` | **Order** — milestone scope, checklists |
| — | `AGENTS.md` (this file) | Process |

Conflict between docs or between docs and code ⇒ **fix the docs first** (via the human),
then make code match. Never silently resolve a spec contradiction in code.

## Repository map

```
cmd/floatnote/        entrypoint + wiring only
internal/config/      paths + constants        ┐
internal/store/       SQLite access, tests     │ headless, unit-tested core
internal/app/         Controller, debounce,    │ (no Fyne imports here)
                      lock/socket              ┘
internal/ui/          Fyne windows/widgets     ┐ UI layer (Fyne imports only here)
internal/hotkey/      OS shims (build tags)    ┘
build/                packaging assets
.github/workflows/    CI
```

Dependency direction: `ui → app → store → config`. Never invert it (e.g. no Fyne imports
in `store`/`app`; no SQL in `ui`).

## Commands (verification gate)

Before declaring any task done, run all of these and paste results in the summary:

```sh
gofmt -l .          # must output nothing
go vet ./...
go test ./...
go build ./...
```

If the change touches windowing/hotkey/drag behavior, also state which numbered items of
the **manual checklist in `ROADMAP.md`** were (or cannot be) exercised — manual items may
be deferred to the human only if explicitly noted.

## Conventions (details in TECH_STACK.md)

- Go 1.27.2, module path `github.com/sambacarlson/floatnote` — never rename the module.
- Standard layout: only `cmd/` + `internal/`. No new top-level code dirs without approval.
- Errors returned and wrapped (`%w` with context); no panics outside `main`; `slog` for all
  logging (`fmt.Print` forbidden in library code).
- Semantic commit prefixes: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `build:`,
  `ci:`, `chore:`. Versions come from **git tags only** (semver, pre-1.0 breaking = minor
  bump) — never invent version constants or CHANGELOG edits unless asked.
- No comments except non-obvious "why".

## Requires explicit approval (do not do unpervised)

1. Changing anything in `VISION.md` / `REQUIREMENTS.md` (especially the Confirmed
   decisions and non-goals) or the decisions tables in other specs.
2. Adding, removing, or upgrading a dependency (Fyne excepted within its pinned major).
3. Schema changes (beyond applying the already-specified v1 migration), new settings keys,
   or file locations of the DB/log/lock/socket.
4. Renaming packages, the module, binary, or app name (A-1).
5. Touching `.github/workflows`, tags/releases, or packaging config.
6. Relaxing an acceptance criterion or removing a manual checklist item.
7. Committing secrets/keys (there are none — keep it that way) or force-pushing.

## Task workflow

1. Identify the requirement IDs (FR/BR/EC/AC) the change implements; if the request
   doesn't map to any, it's a spec change → stop and ask.
2. Read the owning package per the repo map; follow existing patterns in it.
3. Implement + tests (store/app logic: tests mandatory; ui: smoke tests; windowing:
   checklist items).
4. Run the verification gate; report `gofmt/vet/test/build` output and touched checklist
   items.
5. If the implementation revealed a wrong spec assumption, report it — do not "fix" the
   spec yourself.

## Hard limits

- **No code outside the current `ROADMAP.md` milestone scope** (e.g. no markdown rendering,
  no Wayland portal, no signing) until the human moves it in.
- No network calls, telemetry, or auto-updaters — ever (FR-16, VISION non-goals).
- Never delete or recreate a user database automatically (EC-4).
- Keep the app free of theme/appearance systems (vision non-goal).
