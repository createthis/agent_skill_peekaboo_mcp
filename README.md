# peekaboo-mcp skill

An [Agent Skills](https://agentskills.io/specification) skill for driving a macOS desktop
through the **Peekaboo MCP server** — the `window`, `see`, `inspect_ui`, `image`, `type`,
`press`, `click`, `select_text`, `verify_state` tool surface exposed to OpenHands agents.

It is intentionally thin and complementary to the upstream Peekaboo docs (which target the
CLI). This skill captures only the highest-leverage patterns for *efficient* MCP use:

- **Accessibility-first observation.** `inspect_ui` (AX tree) is far lighter than screen
  capture; a capture timeout is not a reason to retry capture.
- **No-sleep observe→act→verify loops.** Dispatch is near-synchronous; a re-observation is
  the right "wait", never `sleep`.
- **Correct handling of "dispatched but unconfirmed" outcomes** so an agent never
  double-types into a real shell.
- **Effect probes without pixels**: caret-delta arithmetic, `select_text` substring checks,
  `verify_state` polling — with a dedicated
  [Terminal.app reference](peekaboo-mcp/references/terminal.md).
- **Reliable command execution in a terminal**: the clipboard + `menu` Paste recipe,
  stuck-continuation recovery, and the `press` key-syntax gotcha
  (`"C-c"` is not a key name; use `{"key":"c","modifiers":["ctrl"]}`).
- **Safety & failure modes**: `mutation_dispatched` retry-safety, `move`/`resize`
  window-ID targeting, locked-screen detection, degraded-bridge signals.
- **System dialogs & file panels**: separate-process ownership, occlusion, the
  "Go to Folder" sheet timeout trap, and panel bypasses — see
  [references/file-dialogs.md](peekaboo-mcp/references/file-dialogs.md).

Nothing in this skill is machine-specific: no hostnames, ports, paths, or usernames. The
target Mac's identity lives in the consuming environment's AGENTS.md / memory.

## Layout

```
peekaboo-mcp/
├── SKILL.md                     # entry point: tool map, fast loop, efficiency + safety rules
└── references/
    ├── terminal.md              # Terminal.app quirks, clipboard+menu Paste, recovery
    └── file-dialogs.md          # NSOpenPanel/NSSavePanel: ownership, occlusion, Go-to-Folder trap
```

To install into an agent workspace, copy (or symlink) the `peekaboo-mcp/` directory into
`.agents/skills/`.

## Contributing

One pattern per entry, each justified by a real session. Keep `SKILL.md` short (it loads
whenever the skill activates); push detail into `references/`.
