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

**CrazyGames counts ALL uploaded files** toward initial load size — not just what the browser downloads on first visit. Include only what the game needs.

---

## Export Settings (export_presets.cfg)

```
[preset.0]
name="Web"
platform="Web"
# IMPORTANT: Only exclude dev-only addons — crazygames SDK must be included!
exclude_filter="addons/auto_reload/*,addons/godot_mcp_editor/*,addons/godot_mcp_runtime/*"
script_export_mode=2             ← bytecode only (no source in pck)

[preset.0.options]
vram_texture_compression/for_desktop=true
vram_texture_compression/for_mobile=false
variant/thread_support=false     ← threads off = no SharedArrayBuffer needed
html/canvas_resize_policy=2      ← responsive canvas
```

**Do not use `exclude_filter="addons/*"`** — it will strip the CrazyGames SDK from the pck and the SDK won't be detected on upload.

**Do not enable thread support** unless you need it — threads require `SharedArrayBuffer` which requires COOP/COEP headers on every page, complicating deployment.

---

## Build Steps

### 1. Export from Godot

Use the **editor GUI** (Project → Export), not headless CLI.

Headless/MCP export can produce a broken .pck (~484KB instead of ~6MB) due to stale import cache. If you move assets between folders, the editor reimports them; headless export skips this.

```bash
# Headless export is NOT reliable — use editor instead
# godot --headless --export-release "Web" path/to/output/index.html
```

### 2. Rename output file

CrazyGames requires the main HTML file to be named `index.html`.

### 3. Verify sizes

```bash
ls -lh *.wasm *.pck *.js
```

### 4. Do NOT add Brotli files for CrazyGames uploads

CrazyGames counts ALL uploaded files toward size. Uploading both `.br` and uncompressed **doubles** your file size in their system. Their CDN does **not** serve `.br` files transparently — if you upload only `.br` files, Godot engine requests the uncompressed names and gets 404s.

**Upload uncompressed files only.** No zipping — files are uploaded directly via the CrazyGames dashboard.

Files to upload:
```
index.html              ← renamed from Experiment.html (or whatever Godot named it)
index.js
index.pck
index.wasm
index.png               ← or custom loading image
index.icon.png
index.apple-touch-icon.png
index.audio.worklet.js
index.audio.position.worklet.js
StudioLogo.png          ← if using custom loading image (copy here manually)
```

---

## Loading Screen Customization

Godot 4 supports only these placeholders in custom HTML shells:
- `$GODOT_HEAD_INCLUDE`
- `$GODOT_PROJECT_NAME`
- `$GODOT_URL` (in some versions)
- `$GODOT_THREADS_ENABLED`

`$GODOT_SCRIPT_ELEMENT` is **NOT a valid placeholder** in Godot 4 — it will appear literally in the browser.

**Recommended approach**: Post-process the exported HTML after export:
1. Export normally
2. Patch the HTML file to replace the splash image src and background color
3. Copy your custom logo file next to the HTML

```python
# Example patch (run after export)
with open("index.html", "r") as f:
    html = f.read()
html = html.replace('src="index.png"', 'src="StudioLogo.png"')
html = html.replace('background-color: #1b1b1b', 'background-color: #2d2d2d')
with open("index.html", "w") as f:
    f.write(html)
```

---

## GodotPlayThing Benchmark (2026-04-28)

| File | Raw | Brotli | Reduction |
|------|-----|--------|-----------|
| index.wasm | 36 MB | 6.2 MB | 83% |
| index.pck  | 6.4 MB | 5.5 MB | 14% |
| index.js   | 309 KB | 66 KB | 79% |
| **Total upload** | **~43 MB** | n/a | Upload uncompressed |

Result: ✅ Under 50MB CrazyGames hard limit. ❌ Above 20MB mobile threshold.

The `.pck` only compresses 14% because WAV audio and pre-compressed textures are already dense. If audio were converted to OGG Vorbis, `.pck` would compress further.

---

## Why .pck Doesn't Compress Well

Godot's `.pck` contains:
- Textures — already compressed as S3TC/ETC2 (nearly incompressible)
- WAV audio — uncompressed PCM (Brotli helps here)
- Scripts — `.gdc` bytecode (moderate compression)
- Scenes — binary format (moderate compression)

**To reduce .pck further:**
1. Convert WAV → OGG Vorbis in Godot's import settings (biggest win, 85–95% audio reduction)
2. Enable `vram_texture_compression/for_mobile=true` for mobile builds (smaller textures)
3. Use `Project → Tools → Optimize Resources` to strip unused imported data

---

## Getting Under 20MB (Mobile Homepage Eligibility)

To reach the 20MB threshold you need BOTH of these:

### Option A: Custom Export Template (disable 3D)

Removes 3D physics engine, 3D rendering, and other unused 3D subsystems from the wasm.
Reduces wasm from ~36MB to ~15–19MB.

1. Download Godot source or use a community template for 2D-only builds
2. Compile with 3D disabled: `scons target=template_release platform=web disable_3d=yes`
3. Set the template in export preset: `custom_template/release = "path/to/template.wasm"`

### Option B: WAV → OGG Vorbis Conversion

1. In Godot editor: select each `.wav` file in FileSystem
2. Import tab → change format to OGG Vorbis, adjust quality (0.7–0.8 is good)
3. Re-export — audio assets will be OGG in the pck

Typical result for a small game: pck shrinks from 6MB to 1–2MB.

### Combined result (estimated)

| Approach | wasm | pck | Total |
|----------|------|-----|-------|
| Current | 36 MB | 6.4 MB | ~43 MB |
| +OGG audio | 36 MB | ~2 MB | ~39 MB |
| +Custom template | ~17 MB | 6.4 MB | ~24 MB |
| Both | ~17 MB | ~2 MB | **~20 MB** ✅ |

---

## CrazyGames SDK Plugin

Plugin location: `addons/crazysdk-godot-4/`

### Known bug in AdModule.gd: adError causes infinite hang

The JavaScript `adError` callback originally emits `{ "state": "started", "error": ... }` which triggers the "wait for second signal" branch in `request_ad_async` — a second `ad_status_change` never fires, so the coroutine hangs forever.

**Fix applied**: Change `adError` in `CrazyGamesBridge.gd` to emit `{ "state": "error", "error": ... }` instead.

### Known bug: request_rewarded_ad always returns false

`request_ad_async` returns a `Dictionary` like `{"state": "finished"}`, but the original code compared it to a plain String `"finished"`, always yielding false.

**Fix applied**: Use `result.get("state", "") == "finished"` instead.

### Known bug: crazy_banner.tscn wrong path

The downloaded plugin has `crazy_banner.tscn` pointing to `res://addons/crazygames/Utils/CrazyBanner.gd` (wrong folder name). Update to `res://addons/crazysdk-godot-4/Utils/CrazyBanner.gd`.

### SDK Detection

Both autoloads must be in `project.godot`:
```ini
[autoload]
CrazyGamesBridge="*res://addons/crazysdk-godot-4/Utils/CrazyGamesBridge.gd"
CrazyGames="*res://addons/crazysdk-godot-4/CrazyGames.gd"
```

The export `exclude_filter` must NOT exclude `addons/crazysdk-godot-4/`. If the SDK `.gdc` files are missing from the pck, the CrazyGames validator will report "SDK not detected".

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

Ads will NOT work in local testing — the ad bridge requires the CrazyGames iframe.
Upload to CrazyGames testing environment to verify ad flows.
