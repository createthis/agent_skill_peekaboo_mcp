# System dialogs & file panels via Peekaboo MCP

`NSOpenPanel` / `NSSavePanel` and app-level alerts are where most automation sessions go
off the rails. They behave differently from ordinary app windows.

## Core facts

- **Open/save panels are owned by a separate process** (e.g. the CoreServices UI agent),
  not the app that opened them. `dialog`/`inspect_ui`/`click` with `app: <the app>` can
  miss the panel or fail to resolve it. Target the panel by its **own window_id** instead:
  `window list` shows it, then `inspect_ui` with that `window_id`.
- **The panel can be occluded** by the app's main window at the same screen coordinates.
  An element click at that point then hits the *other* window. Focus the panel window
  first (`window focus` on the panel's `window_id`) before interacting.

## The "Go to Folder" sheet is the failure point

`dialog` action `file` given a `path` navigates *internally* via the "Go to Folder"
(Cmd-Shift-G) child sheet, and re-triggers it on every call. That transient sheet is where
focus/AX revalidation timeouts concentrate — `press`, `set_value`, and `type` are frequently
refused against it, and `dialog` input is unavailable for it.

Consequences:

- **Don't drive a file panel by `path`.** Navigate by **row clicks** in the column view:
  re-`inspect_ui` before every click (snapshots go stale fast in a panel), pick the folder
  row, confirm.
- **Prefer to bypass the panel entirely** when the app offers an alternative — e.g. a
  "Recently Used" list, a default location, or a programmatic entry point. That removes the
  flakiest surface from the loop.
- If the panel is already open, skip straight to selecting the row + confirming; don't
  re-issue a `path`-based open.

## Fallbacks & cleanup

- If a **modal** panel blocks scripted quit (`osascript -e 'tell application X to quit'`
  hangs because the modal eats it), `killall X` from a terminal works — safe only when the
  only unsaved state is a cancellable panel.
- `dismiss` (or a forced Escape) closes a stuck dialog; `dialog list` first to see what's
  actually there and target the right window.
- Treat the same focus/AX timeout recurring across an *entire* panel (not just the sheet)
  as a **degraded bridge** signal — stop and surface it rather than retrying.

## General dialog workflow

```
window list                     # find the dialog/panel window_id (may be a different process)
inspect_ui (window_id)          # structure: buttons, text fields, rows, sheet state
dialog click (button, window_id)  # or: type into a field, click a row, select a file
inspect_ui (window_id)          # verify the effect before the next step
```

Keep it tight and re-observe each step; panels change their element table on every
navigation, so a consumed snapshot is useless for the next action.
