---
name: peekaboo-mcp
description: "Drive a macOS desktop through the Peekaboo MCP server — list and focus apps/windows, read UI text and elements from the accessibility tree, type, press keys, click, and verify results. Use when a task must operate a Mac: typing into Terminal or any app, reading on-screen text, clicking controls, capturing the screen, or diagnosing why Peekaboo calls are slow or timing out. Optimized for tight observe→act→verify loops with no unnecessary waits."
compatibility: "A Peekaboo MCP server connected to the target macOS machine, with Screen Recording and Accessibility granted there. All names below are the MCP tools the server exposes."
metadata:
  author: createthis
  version: "0.1"
---

# Peekaboo MCP — tight-loop macOS automation

Peekaboo exposes a Mac's windows, accessibility tree, screen capture, and keyboard/pointer input as MCP tools. This skill is about using them **fast**: pick the lightest tool that answers the question, keep the observe→act→verify loop tight, and never wait blindly.

## Mental model (three rules)

1. **Observation and action are separate calls.** Actions are background-safe and do not steal focus unless you explicitly ask (focus/foreground ops).
2. **Snapshots are producer-bound and consumed by mutations.** Every element-targeted action must carry the snapshot ID from the observation made *immediately before it*. Reusing a consumed or stale snapshot is refused pre-dispatch.
3. **Dispatch is not effect.** A successful call means "sent", not "landed". Re-observe to confirm.

## Tool map

| Goal | Tool | Key params / notes |
|---|---|---|
| Running applications | `app` (action `list`) | |
| Windows: IDs, bounds, capability | `window` (action `list`, `app`) | pick an exact `window_id`; rows report `observation_capability` |
| Bring an app/window to the front | `window` (action `focus`) | the one foreground op in a typical loop |
| Read UI text + element IDs (no pixels) | `inspect_ui` | AX only; returns snapshot ID + element table + selection state |
| Full observation: pixels + elements | `see` | raster + AX; the heaviest observation |
| Pixels only | `image` | no element map |
| Type text into the focused field | `type` | `on` = element ID, `snapshot` = fresh ID; trailing `\n` submits in a terminal |
| Keys / chords | `press` | e.g. `["cmd+c"]`; background needs a fresh exact snapshot |
| Click an element | `click` | `on` + `snapshot`; background-safe |
| Substring presence in a text field | `select_text` | found / not-found / ambiguous — a cheap content probe |
| Wait for a condition (no blind sleep) | `verify_state` | polls AX predicates (window exists, element value, enabled) with a hard timeout |
| Set a settable field's value directly | `set_value` | skips typing entirely |
| TCC permissions | `permissions` | one-line preflight before debugging capture |
| Wait (avoid) | `sleep` | exists for clients without a shell; a re-observation is almost always the right "wait" |

## The fast loop

```
window list (app)          → pick the exact window_id (on-screen, topmost)
window focus (app)         → once, up front
inspect_ui (window_id)     → snapshot ps1_…, element IDs, caret/selection position
type (on, snapshot, text)  → or press / click
inspect_ui (window_id)     → immediately: did it land? (caret moved? text present?)
loop.
```

Rules that make it fast:

- **No sleeps between calls.** A re-observation right after a mutation is correct: dispatch is near-synchronous, and the observation takes longer than any sleep you would have added. If the effect is not visible on the first re-observation, observe once more or use `verify_state` — still not a `sleep`.
- **One fresh observation per action.** Observing again is cheaper than debugging a stale-snapshot refusal.
- **Exact targeting.** `window_id` over app name, element ID over coordinates. Fuzzy or app-only targeting is slower and often refused for mutations.

## The big efficiency gains (hard-won)

### 1. Accessibility first, pixels only when you need pixels

`inspect_ui` (AX tree) is the lightest observation and answers most questions: what the terminal shows, what a field contains, whether a control is enabled. `see` and `image` go through screen capture, which is slower and can time out under load or when the machine's capture state is stale. **A capture timeout is not a reason to retry the same capture, and it does not mean the AX path is broken.**

Decision rule: *can I read the answer from the accessibility tree?* If yes → `inspect_ui`. If you genuinely need to look at pixels → `see` (also returns element IDs) or `image` (pixels only). If capture is timing out: fall back to `inspect_ui` for content, and try `image`/`see` with `capture_engine: "classic"` (Core Graphics instead of ScreenCaptureKit) before concluding capture is dead.

### 2. "Not an accepted outcome" does not mean "not typed"

`type` and `press` return an error-shaped outcome when Peekaboo cannot *confirm* the effect — most commonly a text field whose AX value is a truncated, read-only preview (Terminal's shell text area is exactly this). **The input was dispatched. Re-observe; re-type only if observation shows the input is missing.** Blind re-typing is the classic double-input failure mode.

### 3. Probe effects without pixels

- `inspect_ui` reports selection/caret state in text fields: `Text selection <elem>: UTF-16 location L, length N`. Record `L` before typing; if it advanced by roughly the typed characters plus what the app echoed, the input landed exactly once.
- `select_text` is a substring probe: "found" / "not found" / "ambiguous" — enough to confirm a command emitted a particular string, or that an error string did not.
- `verify_state` turns "wait until X is true" into a poll with stable-sample count and a hard timeout — the right tool when something genuinely takes seconds (builds, downloads), not for confirming keystrokes.

### 4. Trim observation payloads

A busy window (many tabs, big menus) can expose hundreds of elements, and every observation carries the full table into your context. When you only need one control or the caret, cap the tree with `max_depth` / `max_elements` / `max_children`. The snapshot ID and selection line come back regardless.

### 5. Preflight once, not every call

`permissions` (Screen Recording, Accessibility, Event Synthesizing) is a one-line check — run it once at the start of a session that will capture or type input. If capture fails but AX works, suspect the capture path (stale capture state, ScreenCaptureKit under load), not permissions. The `window list` row for the target (`observation_capability`: `combined_eligible` / `pixels_only` / `unknown`) is a fast hint about which route a window supports; `unknown` means the inventory itself is stale — refresh it, don't guess.

## Terminal.app specifics

See [references/terminal.md](references/terminal.md) — the shell text area's truncated read-only value, caret-delta verification, tab-title CWD, and error-string probes. Screenshots are rarely needed for terminal work.

## Scope

This skill covers efficient MCP-tool use in tight loops. For exhaustive flags, the full key-name vocabulary, and protocol/bridge internals, read the upstream Peekaboo documentation — this skill deliberately does not duplicate it.
