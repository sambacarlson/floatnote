# VISION — FloatNote

## Problem

Short-lived text — a command you're about to paste, a snippet you're comparing against, a
temporary todo — gets lost when it lives in a full note app. Using one means switching away
from the app you're actually working in, finding the right note, and switching back. People
who live in terminals and editors want their scratch text **visible over the current work**,
not one Alt-Tab away.

## Target users

- **Primary:** developers and power users on macOS and Linux (X11) who keep short scratch
  text while working in other applications.
- **Secondary:** anyone who wants a digital sticky pad that stays on top.

## Solution

FloatNote is a deliberately minimal floating note app:

- A small **borderless, always-on-top text window** that floats above other applications.
- **Drag anywhere** by its header; **resize** from a corner grip.
- **Minimize to a floating bubble** — a small draggable dot that stays on screen; click it to
  restore the note.
- **Global hotkey** (macOS + X11) toggles the note window from anywhere.
- **Multiple notes**, switched from a compact dropdown in the window header; titles are
  derived from the first line of the note.
- **Plain text only**, always-editable, **autosaved** to a local SQLite database.
- No themes, no formatting, no sync, no accounts — that restraint is the product.

## Goals

- **G1:** The note is one hotkey press away and floats above normal windows on macOS and
  Linux/X11.
- **G2:** Zero-friction editing: focus, type, done — autosaved without a save button.
- **G3:** Window and bubble positions/sizes survive restarts and monitor changes.
- **G4:** Installable artifacts: unsigned macOS `.app` and a Linux AppImage.
- **G5:** Maintained spec-first: a coding agent (or new contributor) can implement and
  verify changes using only the documents in this repository.
- **G6:** Lightweight: single process, single binary, one database file, no network.

## Non-goals (v0.x)

- Rich text, Markdown rendering, images, attachments.
- Themes or appearance settings.
- Sync, collaboration, cloud, accounts, telemetry.
- Windows support.
- System tray icon (the floating bubble replaces it).
- Encryption at rest, search, tags, folders.
- Guaranteed always-on-top or global hotkeys under Wayland (documented degradation, not
  an MVP deliverable).
- Code signing / notarization (deferred past MVP; unsigned-install workaround documented).

## Success measures

| # | Measure |
|---|---|
| S1 | All acceptance criteria in `REQUIREMENTS.md` pass the manual checklist on macOS and Ubuntu (X11). |
| S2 | Cold start to interactive window < 1 second (informal timing). |
| S3 | Typing followed by `kill -9` loses at most 500 ms of text (the debounce window). |
| S4 | A behavior change (e.g. autosave debounce) can be located and implemented in < 30 minutes using only these specs. |
| S5 | Install path works: `.app` opens after the documented `xattr` command; AppImage runs on a clean Ubuntu LTS. |
| S6 | No network syscalls from the app after startup (verified by inspection / sandbox). |
