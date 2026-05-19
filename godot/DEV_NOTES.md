# Godot Dev Notes

Learnings from shipping Godot 4.x web games.

---

## Testing After Code Changes

After any `.gd` file change, run headless to catch parse/script errors:
```powershell
godot --path . --headless --quit-after 5 2>&1 | Out-String
```
Check output for `Parser Error`, `SCRIPT ERROR`, `ERROR`. The `| Out-String` is required — without it PowerShell swallows stderr and you'll see no errors even when they exist.

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

### Web audio
`AudioStreamOggVorbis.load_from_file(ProjectSettings.globalize_path("res://..."))` fails on web — the globalized path doesn't exist in the browser sandbox. Use `load("res://...") as AudioStreamOggVorbis` instead, which goes through Godot's VFS.

Also set output latency on web or audio may not initialize:
```gdscript
if OS.has_feature("web"):
    AudioServer.set("driver/output_latency", 50)
```

### Font symbols on web
Special Unicode/emoji characters don't render in Godot web exports — they show as hex codepoints or boxes. Sweep the entire UI before shipping and replace all emoji with plain text or sprite-based alternatives.

### Escape key on web
CrazyGames uses Escape to exit fullscreen. Guard all `ui_cancel` handlers:
```gdscript
if OS.has_feature("web"):
    return
```
Add a visible pause button to the HUD for web builds as replacement.

### Async lambdas
GDScript 4 supports `await` inside lambdas. The lambda becomes a coroutine. Callers that don't await it fire-and-forget — the coroutine continues asynchronously.

### Engine.time_scale vs pausing
`get_tree().paused = true` pauses all nodes with default process mode.
`Engine.time_scale = 0` only slows physics/process, doesn't pause.
Timers created with `create_timer(..., true, false, true)` run even when paused (process_always=true, ignore_time_scale=true).

### iOS audio
iOS suspends AudioContext on background. Must call `audioContext.resume()` within a user-triggered event. Test on iOS before shipping.
