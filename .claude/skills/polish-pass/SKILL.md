---
name: polish-pass
description: Run a structured Godot game polish pass for GodotPlayThing. Covers game feel/juice, particles, audio (flags missing SFX as a list), UI, and visuals. Implements changes directly with user approval. Expects multiple iterations before export. Use when user says "do a polish pass", "polish the game", "polish pass", or asks to improve game feel/quality.
---

# Polish Pass

Structured polish pass for this project. Asks clarifying questions, scans the project, works through each category, implements with approval, then summarizes and loops.

## Trigger Phrases
- "do a polish pass"
- "polish pass"
- "polish the game"
- "add juice"

## Resources
- Shader library: https://godotshaders.com/shader/ — always check here before writing custom shaders.

---

## Step 1: Clarifying Questions

Ask before touching anything:

1. **Focus area** — Any specific category feeling most lacking? Or full pass?
2. **Recent changes** — What changed since the last pass (new mechanics, levels, enemies)?
3. **Pain points** — What feels bad when you play it right now?

---

## Step 2: Checklist

Work through each category. For each item:
- Check current state by reading scripts + inspecting scenes via MCP tools
- Report: ✅ Done / 🟡 Partial / ❌ Missing
- For ❌/🟡: propose what to do, wait for approval, implement
- Test via `mcp__godot__editor-run` after each batch

---

### 1. Game Feel / Juice

- [ ] **Screen shake** — On hits, walls, deflects. Use Camera2D offset + Tween.
- [ ] **Hit stop / freeze frame** — Brief `Engine.time_scale = 0.05` + Timer on impactful hits.
- [ ] **Squash & stretch** — Scale tween on jump, land, shoot, hurt.
- [ ] **Camera smoothing** — `position_smoothing_enabled` on Camera2D, or lerp in `_process`.
- [ ] **Damage flash** — Modulate color tween on hit (white flash or blink).
- [ ] **Input buffering** — Buffer action input ~0.1–0.15s to forgive mistimed presses.
- [ ] **Death/respawn feel** — Slow-mo + effect on death, smooth fade on respawn.

### 2. Particle FX

- [ ] **Impact particles** — `GPUParticles2D` / `CPUParticles2D` on bullet hits, collisions.
- [ ] **Death/destroy effect** — Burst on enemy or object destruction.
- [ ] **Pickup/reward effect** — Sparkle or pop on collecting items.
- [ ] **Projectile trail** — Line2D or particles trailing fast-moving objects.
- [ ] **Environmental ambience** — Dust, debris, or atmosphere particles for world feel.

### 3. Audio — Flag Missing SFX

Do not implement audio assets. Scan all interaction points in scripts and scenes. For each action that should have a sound, check if an `AudioStreamPlayer` is wired with an actual stream.

Report the full list of missing SFX to the user — they source the assets, you wire them up once provided.

Also flag audio infra gaps:
- Audio bus structure (Master → Music, SFX buses)
- Pitch variation on repeated SFX (`pitch_scale` ± 0.1)
- UI sounds (button hover, click, transition)
- Volume settings exposed to player

### 4. UI / HUD

- [ ] **Scene transitions** — Fade in/out via `ColorRect` + `AnimationPlayer` or Tween overlay.
- [ ] **Button animations** — Hover scale-up, press scale-down via Tween.
- [ ] **HUD reactivity** — Health bar pulses when low, score pops on increment, combo animates.
- [ ] **Consistent palette** — Fonts, colors, icon style coherent across all UI scenes.
- [ ] **Victory / defeat screen** — Animated, shows stats, clear CTA (retry / next level / menu).
- [ ] **Pause menu** — Accessible mid-game, dims background, has resume/restart/quit.

### 5. Visuals / Rendering

- [ ] **Post-processing** — `WorldEnvironment`: glow, color correction, optional vignette.
- [ ] **Shaders** — Check https://godotshaders.com/shader/ for hit flash, transitions, outlines, special effects before writing custom ones.
- [ ] **Lighting pass** — Add `PointLight2D` where it adds atmosphere.
- [ ] **Parallax / background depth** — `ParallaxBackground` layers if applicable.
- [ ] **Animation polish** — Key animations feel weighty (anticipation, follow-through).

---

## Step 3: End-of-Pass Summary

After completing all categories, output:

```
POLISH PASS SUMMARY
===================
✅ Implemented: [list]
🟡 Partial:     [list with what remains]
❌ Skipped:     [list with reason]
🔊 SFX needed:  [full list for user to source]

Ready for another iteration or proceed to Export Settings?
```

Expect 2–3 iterations before the game is ready for export.
