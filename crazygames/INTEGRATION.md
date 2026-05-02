# CrazyGames Integration Guide

Full integration reference for Godot 4.x games targeting CrazyGames (web export).
Source: https://docs.crazygames.com/

---

## SDK Setup

Install the official CrazyGames Godot plugin:
https://store.godotengine.org/asset/crazygames/crazysdk/

Add a `CrazySDK` autoload wrapper that stubs all calls when the plugin is absent (local dev, Steam, mobile builds). This means integration code works across all platforms without ifdefs everywhere.

Godot method namespace:
- `CrazyGames.Game.*` — gameplay lifecycle, settings
- `CrazyGames.Ad.*` — ads
- `CrazyGames.User.*` — auth, system info
- `CrazyGames.Data.*` — cloud save

> Loading tracking (`loading_start`/`loading_stop`) is NOT needed in Godot — the export handles it automatically.

---

## Launch Tiers

| Tier | Ads | Requirements |
|------|-----|--------------|
| Basic Launch | Disabled | gameplay_start event only |
| Full Launch | Enabled | gameplay start/stop + ads + applicable modules |

Games enter Basic Launch first for a 2-week KPI window, then graduate to Full Launch.

---

## Phase 1 — Mandatory Core

### Gameplay Events (required for Full Launch)

```gdscript
CrazyGames.Game.gameplay_start()  # player enters active play
CrazyGames.Game.gameplay_stop()   # any pause, menu, game over, or level transition
```

Call `gameplay_start` when:
- Player first enters gameplay (first aim, first click, etc.)
- Player resumes from pause
- Player continues after a level-up choice
- Player continues after a rewarded ad

Call `gameplay_stop` when:
- Game over screen appears
- Win/level complete screen appears
- Level-up choice screen appears
- Pause menu opens
- Level transition begins

### Settings Compliance

Check on startup and respond to changes:
```gdscript
var settings = CrazyGames.Game.get_game_settings()
if settings.get("muteAudio", false):
    # mute all game audio via AudioServer bus
    AudioServer.set_bus_mute(AudioServer.get_bus_index("Master"), true)
if settings.get("disableChat", false):
    # hide any in-game chat UI
    pass
```

### Escape Key (web only)

CrazyGames uses Escape to exit fullscreen. Do NOT intercept it on web:
```gdscript
func _unhandled_input(event: InputEvent) -> void:
    if OS.has_feature("web"):
        return
    if event.is_action_pressed("ui_cancel"):
        # pause menu etc.
```

On web, add a visible pause button to the HUD instead.

---

## Phase 2 — Ads

### Midgame Ads

Show at natural breaks only (game over, level transition). Never during active gameplay.

```gdscript
# Caller must call gameplay_stop() first
AudioManager.mute_all()
await CrazyGames.Ad.request_ad_async("midgame")
AudioManager.unmute_all()
# Caller resumes gameplay_start() if appropriate
```

Rules:
- SDK auto-enforces 3-minute cooldown between midgame ads
- Game must remain fully playable with adblocker active (never gate content on ad delivery)
- Handle `unfilled`, `adblock`, `adCooldown` errors gracefully

### Rewarded Ads

Player-initiated, in exchange for in-game benefit. Must have a non-ad alternative.

```gdscript
var result = await CrazyGames.Ad.request_ad_async("rewarded")
var granted = result is Dictionary and result.get("state", "") == "finished"
# Only grant reward if granted == true
# NOTE: request_ad_async returns a Dictionary, not a String — comparing result == "finished" always evaluates false
```

Rules:
- Never chain multiple rewarded ads for one reward
- Button must be consistently visible and optional
- Show a video icon on the button
- Never combine midgame + rewarded for "continue playing" flow between levels
- Only reward on `adFinished`; deny on `adError`

### Banner Ads

Only on useful screens open for at least 5 seconds. Never during active gameplay.

```gdscript
CrazyGames.Banner.request_banner({id = "banner-container", width = 300, height = 250})
```

Available sizes: 728×90, 300×250, 320×50, 468×60, 320×100
Refresh minimum: 30 seconds. Session max: 120 refreshes per size.
Clear banners when hiding them.

---

## Phase 3 — Cloud Save (Data Module)

Replaces localStorage. Syncs across devices when user is logged in.

```gdscript
CrazyGames.Data.data_set_item("key", value)
CrazyGames.Data.data_get_item("key")
CrazyGames.Data.data_has_key("key")
```

- 1MB cap per user
- Always read existing data before writing (prevent overwrite)
- Must enable "Progress Save" toggle in developer dashboard
- Guest users use localStorage; data auto-transfers on login

---

## Phase 4 — User Auth

Only required if game has in-game accounts or user-specific progression.

```gdscript
var user = CrazyGames.User.get_user()  # null if not logged in
CrazyGames.User.show_auth_prompt()     # login/register modal
# Register auth listener for mid-session logins:
CrazyGames.User.add_auth_listener(callback)
```

Rules:
- Allow guest play by default
- No external logins (Facebook, Google, email) allowed
- On login: auto-register new users, auto-login returning users
- Handle account switching via auth listener

User data available: `username`, `profilePictureUrl`, `__dangerousUserId` (client-only).
For server-side auth use `CrazyGames.User.get_user_token()` (JWT, 1hr lifetime).

---

## Phase 5 — Optional Features

### Happytime
```gdscript
CrazyGames.Game.happytime()  # triggers confetti on CrazyGames site
```
Use on major achievements (boss kill, new high score). Use sparingly.

### System Info
```gdscript
var info = CrazyGames.User.get_system_info()
# info.device_type → "desktop" | "tablet" | "mobile"
# info.locale → "en-US" etc.
# info.country → "US" etc.
```
Use locale for language detection; fall back to English if language not supported.

### Game Context
```gdscript
CrazyGames.Game.set_game_context({level = 3, score = 450})
CrazyGames.Game.clear_game_context()
```
Attaches metadata to player bug reports. Clear when context changes significantly.

### Leaderboards
Configure encryption key + metric type in developer dashboard first.
```gdscript
CrazyGames.Leaderboard.*
```

---

## Platform Fit Requirements

### Content
- PEGI 12 / 13+ audience — no adult content
- English required as fallback language
- No App Store links, no cross-promotion of external games
- No custom fullscreen button (CrazyGames provides one)

### Gameplay
- New players must reach gameplay in 1 click maximum
- Tutorials must be gameplay-integrated, visual-first
- Keyboard/mouse control overlays required (show what buttons do what)
- Adapt bindings for QWERTY and AZERTY layouts
- Physics must be framerate-independent (always multiply by delta)

### Visual / Audio
- UI must be legible at 800×450 (mobile) through 1920×1080 (desktop)
- Consistent art style throughout (no mixing of styles/resolutions)
- Audio properly leveled

---

## Game Covers (required assets)

| Size | Dimensions |
|------|-----------|
| Landscape | 1920×1080 (16:9) |
| Portrait | 800×1200 (2:3) |
| Square | 800×800 (1:1) |

Rules: artistic representation of game world, main character prominent, game title only (no extra text), no screenshots, no borders, no store logos.

Preview video: 15–20 sec, 1080p, no audio, max 50MB.

---

## Post-Launch KPIs (2-week evaluation window)

| Metric | Target | What it measures |
|--------|--------|-----------------|
| Conversion rate | ≥ 80% | Players who engage 1+ min after clicking Play |
| Avg session length | 10+ min | Core loop engagement |
| Day 1 retention | 10–15% | Return visits next day |

Games meeting KPIs graduate to Full Launch with monetization.

Conversion rate is most actionable: load fast + get to fun immediately. Keep initial download under 20MB for homepage eligibility.

---

## Export Checklist

- [ ] Initial download ≤ 50MB (≤ 20MB for mobile homepage eligibility)
- [ ] Total files ≤ 1,500
- [ ] Relative paths only (no absolute paths in exported HTML)
- [ ] Brotli compression on `.wasm` + `.pck` (serve with `Content-Encoding: br`)
- [ ] Sitelock: whitelist `*.crazygames.com` and regional TLDs if using CSP
- [ ] Test on Chromebook-equivalent (4GB RAM, no discrete GPU)
- [ ] Chrome + Edge functional; Safari recommended
- [ ] iOS audio: call `audioContext.resume()` on first user touch
- [ ] Escape key not intercepted on web builds
- [ ] `gameplay_start()` fires before first gameplay frame
- [ ] Audio muted during ads, unmuted after

---

## Platform Policies

- No exclusivity — ship on Steam/mobile/itch simultaneously, still earn on CrazyGames
- Basic Launch first, ads disabled for KPI measurement window
- No pay-to-feature — visibility is purely engagement-driven
- Rejections are recoverable — fix and resubmit
- €100 payout minimum, carries over month-to-month
- Portrait games allowed with developer-managed letterboxing

---

## ByteBrew Analytics

CrazyGames official analytics partner. Free — covers sessions, custom events, A/B testing, remote config.
Docs: https://docs.crazygames.com/resources/partners/#bytebrew-analytics
ByteBrew docs: https://docs.bytebrew.io/sdk/godot

**For Godot web exports, use the JavaScript SDK** (not the Godot native plugin — that's mobile/desktop only).

### SDK

CDN: `https://unpkg.com/bytebrew-web-sdk@1.0.1/dist/ByteBrewSDK.js`

Add to exported `index.html` `<head>` (Godot writes this file on export — patch it post-export or use a custom HTML shell):

```html
<script src="https://unpkg.com/bytebrew-web-sdk@1.0.1/dist/ByteBrewSDK.js"></script>
```

### Initialization

Call from GDScript after SDK loads, e.g. at the end of `_ready()` in the CrazySDK autoload:

```gdscript
if OS.has_feature("web"):
    JavaScriptBridge.eval("""
        ByteBrew.initializeByteBrew('GAME_ID', 'SDK_KEY', '1.0.0');
    """)
```

### Custom Events

```gdscript
func track(event_name: String, params: Dictionary = {}) -> void:
    if not OS.has_feature("web"):
        print("[ANALYTICS] ", event_name, " ", params)
        return
    var params_json = JSON.stringify(params)
    JavaScriptBridge.eval("ByteBrew.newCustomEvent('%s', %s);" % [event_name, params_json])
```

Event name rules: no spaces, periods, or colons — use underscores.

### Recommended events for progression tracking

| Event | When | Key params |
|-------|------|------------|
| `level_started` | First shot fired | `level` (scene basename) |
| `level_midpoint` | 50% timer elapsed | `level`, `score`, `active`, `passives` |
| `level_complete` | Timer runs out | `level`, `score`, `active`, `passives` |
| `player_died` | Game over | `level`, `score`, `time_survived`, `active`, `passives` |

### CrazyGames data partner

Invite `analytics@crazygames.com` in ByteBrew dashboard → Data Partners.
