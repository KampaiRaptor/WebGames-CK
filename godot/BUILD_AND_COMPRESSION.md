# Godot Web Build & Compression Guide

Targeting CrazyGames (and general web hosting). Covers export, Brotli compression, and size budgets.

---

## CrazyGames Size Requirements

| Limit | Threshold | Impact |
|-------|-----------|--------|
| Hard limit | 250MB total, 1500 files | Submission rejected |
| Initial download | ≤ 50MB | Required for any launch |
| Initial download | ≤ 20MB | **Required for mobile homepage eligibility** |

"Initial download" = everything loaded before the first `gameplay_start()` event fires.

---

## Export Settings (export_presets.cfg)

```
[preset.0]
name="Web"
platform="Web"
exclude_filter="addons/*"        ← keep addons out of the build
script_export_mode=2             ← bytecode only (no source in pck)

[preset.0.options]
vram_texture_compression/for_desktop=true
vram_texture_compression/for_mobile=false
variant/thread_support=false     ← threads off = no SharedArrayBuffer needed
html/canvas_resize_policy=2      ← responsive canvas
```

**Do not enable thread support** unless you need it — threads require `SharedArrayBuffer` which requires COOP/COEP headers on every page, complicating deployment.

---

## Build Steps

### 1. Export from Godot

```bash
# Via Godot CLI (headless)
godot --headless --export-release "Web" path/to/output/index.html

# Or use MCP tool: mcp__godot__export-run
```

### 2. Brotli Compress

Brotli typically achieves 80%+ compression on wasm. Apply after every export.

```bash
brotli -q 11 --force Experiment.wasm -o Experiment.wasm.br
brotli -q 11 --force Experiment.pck  -o Experiment.pck.br
brotli -q 11 --force Experiment.js   -o Experiment.js.br
```

`-q 11` = max quality (slowest, best ratio). Fine for CI since you only compress once per build.

Install brotli: `winget install Google.Brotli` or comes with most Linux distros.

### 3. Verify sizes

```bash
ls -lh *.wasm *.wasm.br *.pck *.pck.br *.js *.js.br
```

---

## GodotPlayThing Benchmark (2026-04-28)

| File | Raw | Brotli | Reduction |
|------|-----|--------|-----------|
| Experiment.wasm | 36 MB | 6.2 MB | 83% |
| Experiment.pck  | 6.4 MB | 5.5 MB | 14% |
| Experiment.js   | 309 KB | 66 KB | 79% |
| **Total initial download** | **~42.7 MB** | **~11.8 MB** | **72%** |

Result: ✅ Under 20MB mobile threshold. ✅ Well under 50MB hard limit.

The `.pck` only compresses 14% because WAV audio and pre-compressed textures are already dense. If audio were converted to OGG Vorbis, `.pck` would compress further.

---

## Why .pck Doesn't Compress Well

Godot's `.pck` contains:
- Textures — already compressed as S3TC/ETC2 (nearly incompressible)
- WAV audio — uncompressed PCM (Brotli helps here)
- Scripts — `.gdc` bytecode (moderate compression)
- Scenes — binary format (moderate compression)

**To reduce .pck further:**
1. Convert WAV → OGG Vorbis in Godot's import settings (biggest win)
2. Enable `vram_texture_compression/for_mobile=true` for mobile builds (smaller textures)
3. Use `Project → Tools → Optimize Resources` to strip unused imported data

---

## Local Testing Server (serve.py)

Godot web export requires COOP/COEP headers and benefits from Brotli serving.
The serve.py in the build directory handles both:

```python
# Automatically serves .wasm.br / .pck.br / .js.br when:
#   1. The .br file exists
#   2. The request has "br" in Accept-Encoding
# Falls back to uncompressed if .br isn't found.
```

Run: `python serve.py` → http://localhost:8080

---

## CrazyGames Upload

Upload the full build directory contents including `.br` files.
CrazyGames' CDN will serve them — their infrastructure supports `Content-Encoding: br`.

Rename `Experiment.html` → `index.html` before uploading (CrazyGames requires this).

Files to include in upload zip:
```
index.html              ← renamed from Experiment.html
Experiment.js
Experiment.js.br
Experiment.pck
Experiment.pck.br
Experiment.wasm
Experiment.wasm.br
Experiment.png          ← splash/loading image
Experiment.icon.png
Experiment.apple-touch-icon.png
Experiment.audio.worklet.js
Experiment.audio.position.worklet.js
```

Do NOT include `serve.py` or `.gz` files (not needed if `.br` is present).

---

## CrazyGames SDK Plugin

Plugin location: `addons/crazysdk-godot-4/`
Asset store: https://store.godotengine.org/asset/crazygames/crazysdk

**Known issue with the downloaded plugin:** `sdk_load.gd` hardcodes paths to
`res://addons/crazygames/` instead of the actual folder name. Fix:

```gdscript
# sdk_load.gd — correct paths
func _enter_tree():
    if not ProjectSettings.has_setting("autoload/CrazyGamesBridge"):
        add_autoload_singleton("CrazyGamesBridge", "res://addons/crazysdk-godot-4/Utils/CrazyGamesBridge.gd")
    if not ProjectSettings.has_setting("autoload/CrazyGames"):
        add_autoload_singleton("CrazyGames", "res://addons/crazysdk-godot-4/CrazyGames.gd")
```

Also add both autoloads directly to `project.godot` as a fallback so they survive editor reloads
even if the plugin's `_enter_tree` hasn't fired yet.
