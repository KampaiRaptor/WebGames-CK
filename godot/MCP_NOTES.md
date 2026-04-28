# Godot MCP + Claude Code Notes

Learnings from using Claude Code with the Godot MCP server in GodotPlayThing project.

---

## Setup

The project uses two MCP-related plugins:
- `addons/godot_mcp_editor/` — editor plugin (runs in Godot editor)
- `addons/godot_mcp_runtime/` — runtime autoload (runs in-game)

Both are registered in `project.godot` as editor plugins and autoloads respectively.

The MCP server connects via `mcp__godot__*` tools exposed to Claude Code.

---

## Available MCP Tools

Key tools used in development:

| Tool | Use |
|------|-----|
| `mcp__godot__editor-run` | Start the game from Claude |
| `mcp__godot__editor-stop` | Stop running game |
| `mcp__godot__editor-debug-output` | Read print/error output from running game |
| `mcp__godot__lsp-diagnostics` | Get GDScript type errors |
| `mcp__godot__scene-nodes` | Inspect scene tree |
| `mcp__godot__scene-node-properties` | Read node property values |
| `mcp__godot__script-modify` | Edit scripts via MCP (use with care) |
| `mcp__godot__project-info` | Get project settings summary |

Full tool catalog: `mcp__godot__tool-catalog`

---

## Workflow Notes

### Testing Changes
Per project CLAUDE.md:
1. After changes: stop any running instance, then `mcp__godot__editor-run`
2. Check output: `mcp__godot__editor-debug-output` + grep for `[TEST]`/`[ERROR]`/`Parser Error`
3. Don't rely on LSP diagnostics alone — verify in running game

### Temporary print traces
Add `print("[TEST] some value: ", var)` during development, remove after confirming.

### Parser Errors
GDScript parser errors show in `editor-debug-output` with "Parser Error" prefix.
Always check after modifying `.gd` files before reporting a task done.

---

## Known Gotchas

### Autoloads and plugin availability
When referencing optional autoloads (e.g. CrazyGames SDK), use dynamic node lookup to avoid parser errors when the plugin isn't installed:
```gdscript
var cg = get_node_or_null("/root/CrazyGames")
if cg and "Game" in cg:
    cg.Game.gameplay_start()
```
Direct static reference (`CrazyGames.Game.gameplay_start()`) causes a parse error if the autoload doesn't exist.

### Async lambdas
GDScript 4 supports `await` inside lambdas (Callables). The lambda becomes a coroutine.
Callers that don't await it just fire-and-forget — the coroutine continues asynchronously.
This is useful for button press handlers that need to await an ad result before continuing.

### Engine.time_scale and pausing
`get_tree().paused = true` pauses all nodes with default process mode.
`Engine.time_scale = 0` only slows physics/process, doesn't pause.
The game uses `time_scale` for hit-stop and aim slow-motion, and `paused` for actual game pauses.
Timers created with `create_timer(..., true, false, true)` run even when paused (process_always=true, ignore_time_scale=true).

### Web audio (iOS)
iOS suspends AudioContext on background. Must call `audioContext.resume()` within a user-triggered event. Godot's web export handles this in some versions — test on iOS before shipping.

### Escape key on web
CrazyGames uses Escape to exit fullscreen. Guard all `ui_cancel` handlers:
```gdscript
if OS.has_feature("web"):
    return
```
Add a visible pause button to the HUD for web builds as replacement.

---

## Project-Specific Autoloads

| Name | Script | Purpose |
|------|--------|---------|
| `AudioManager` | `scripts/audio_manager.gd` | All sound playback + mute/unmute |
| `GameState` | `scripts/game_state.gd` | Persist ability loadout between scenes |
| `CameraShake` | `scripts/camera_shake.gd` | Screen shake controller |
| `CrazySDK` | `scripts/crazy_sdk.gd` | CrazyGames SDK wrapper (stubs when plugin absent) |

---

## Claude Code Tips for Godot Projects

- Use `mcp__godot__lsp-diagnostics` to catch type errors before running
- Use `Grep` to find all call sites before renaming a function
- Use `mcp__godot__editor-debug-output` after every test run — don't assume it worked
- Read `project.godot` before adding autoloads to understand existing structure
- The `@onready` annotation requires the node to exist in the scene — check scene files if errors appear
