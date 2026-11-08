# Terminal.app via Peekaboo MCP

Terminal's shell pane is an `AXTextArea` (label "shell"). It works, but it has three quirks that shape the fastest workflow.

## Quirks

1. **Its AX value is a truncated, read-only preview of the top of the scrollback buffer.** It shows the session start, not the current prompt line. Therefore:
   - `type`/`press` into it will almost always return a "not an accepted outcome" / unconfirmed error — **the dispatch happened**; this is the confirmed-effect check failing, not an input failure.
   - You cannot read the latest command output from the value preview directly.
2. **The caret is visible.** Every observation reports `Text selection <elem>: UTF-16 location L, length N`. `L` is monotonically non-decreasing as the buffer grows.
3. **The window title and the selected tab title carry the CWD** (e.g. `~/some/path — -bash`), and the selected tab's element has `value: "true"`. This is a cheap CWD read without running anything.

## The fast workflow

```
window list (app: "Terminal")          # find the window; the focused/key window is the one
                                       # with observation_capability "combined_eligible"
window focus (app: "Terminal")         # once
inspect_ui (window_id)                 # note: snapshot ID, shell elem ID, caret L0
type (on: <shell elem>, snapshot, "pwd\n")
inspect_ui (window_id)                 # caret L1
```

### Reading `pwd` without pixels

- **Caret delta:** `L1 - L0` should equal `len("pwd\n")` + length of the echoed output line (path + newline) + length of the next prompt line. If it matches, the command ran exactly once and you can compute the total output size without reading it.
- **Confirm the exact path:** run `select_text` on the shell element with the *full path you expect* (from the tab title: expand `~` to the home path). A "found" (or "ambiguous" = found multiple times, e.g. a repeat of the same command) confirms the string is in the buffer. A "not found" means your guess was wrong — do not re-type input, probe differently.
- **Negative probe for errors:** `select_text` with `No such file or directory` (or `command not found`). "Not found" after a command is a successful error-free confirmation.

### Running `ls -al <path>`

Same pattern. After dispatch, re-observe:
- **Caret delta** ≈ command length + listing output + prompt. A directory listing that overflows the screen also changes the scroll area (content height grows, a scrollbar element appears) — visible in the element table as new/changed "scroll area"/scrollbar entries.
- **Error probe** as above.

A small delta (command + short error + prompt, no scrollbar growth) means the command failed — probe for the error string to see which.

## Multiple Terminal windows/tabs

`window list` shows every Terminal window, and the tab bar is in the AX tree: each tab is a checkbox-like element whose `help` is the tab's title; `value: "true"` marks the selected tab. The topmost window (or the one you focused) is what input targets. When several full-screen windows are stacked at identical bounds, **the one with `observation_capability: "combined_eligible"` is the current key window** — the others report `pixels_only` with `no_matching_accessibility_window`. Target the key window's ID explicitly; app-level targeting will be ambiguous.

## When you actually need pixels

`select_text` + caret deltas cover almost everything. Reach for `see`/`image` only when you must judge visual state (colors, layout, a rendered figure). If capture times out, switch to `capture_engine: "classic"` for one attempt before falling back entirely to the AX workflow above.
