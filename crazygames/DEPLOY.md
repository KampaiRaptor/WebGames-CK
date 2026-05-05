# CrazyGames — Godot 4 Deploy Workflow

## Export preset setup

Use a dedicated export preset for CrazyGames (separate from any other web preset):

```
export_filter = "all_resources"
exclude_filter = "addons/auto_reload/*,addons/godot_mcp_editor/*,addons/godot_mcp_runtime/*"
export_path = "<your_build_dir>/index.html"
html/head_include = '<script src="ByteBrewSDK.js"></script>'
```

**Must export from the Godot editor** (Project → Export → your preset → Export Project). Headless export skips resource import and produces a broken build.

---

## Post-export script

Run after every export. Minimum it must do:

1. Copy `ByteBrewSDK.js` (self-hosted) into the build directory
2. Patch `index.html` to replace any CDN URLs with local paths (CrazyGames CSP blocks unpkg.com and other CDNs)

Example PowerShell:
```powershell
$BUILD = "D:\Export\YourGame\Build"
$PROJECT = "D:\YourProject"

Copy-Item "$PROJECT\web\ByteBrewSDK.js" "$BUILD\ByteBrewSDK.js" -Force

$html = Get-Content "$BUILD\index.html" -Raw -Encoding UTF8
$html = $html -replace 'https://unpkg\.com/bytebrew-web-sdk@[^"]+/dist/ByteBrewSDK\.js', 'ByteBrewSDK.js'
Set-Content "$BUILD\index.html" $html -Encoding UTF8 -NoNewline
```

---

## File sizes and compression

Raw wasm is ~36 MB, pck varies. **No manual compression needed.** CrazyGames CDN compresses files automatically on their side. Dashboard "Total size" shows uncompressed bytes; "Load size" shows what players actually download (~14-15 MB for a typical Godot 4 game).

Mobile homepage eligibility requires load size ≤ 20 MB — CDN compression is enough to meet this for standard Godot 4 exports.

---

## Files to upload

```
index.html
index.js
index.pck
index.wasm
index.png
index.icon.png
index.apple-touch-icon.png
index.audio.worklet.js
index.audio.position.worklet.js
ByteBrewSDK.js
```

Upload via the CrazyGames developer dashboard. No zip required. No `.br` files needed.

---

## Verify after upload

- Load size on dashboard should be ~14-15 MB (CDN-compressed), not raw ~43 MB
- Open the game in CrazyGames preview and check browser console for errors
- Confirm ByteBrew events firing (Custom Workspace → Mechanics in ByteBrew dashboard)
