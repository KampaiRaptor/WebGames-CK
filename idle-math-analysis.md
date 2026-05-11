# Idle Game Math — Theory vs Blackhole

Source articles: [Part I](https://www.sirpinski.com/the-math-of-idle-games-part-i/) | [Part II](https://www.sirpinski.com/the-math-of-idle-games-part-ii/) | [Part III](https://www.sirpinski.com/the-math-of-idle-games-part-iii/)

---

## Part I — The Standard Model

### Core Vocabulary

| Term | Definition |
|------|-----------|
| Primary Currency | The main number the player grows |
| Generator | Produces primary currency at a measured rate |
| Multiplier | Upgrade that boosts generator output |
| Prestige | Reset with persistent bonus; the meta-loop |

### Cost Formula (exponential)

```
cost_next = base_cost × growth_rate ^ owned
```

### Production Formula (polynomial)

```
production = base_production × owned × multipliers
```

**Key insight:** Exponential costs always overtake polynomial production. This natural bottleneck forces prestige. It's the game's core tension.

### Bulk Purchase Formulas

```
cost_of_n = b × r^k × (r^n - 1) / (r - 1)

max_buyable = floor( log_r( c(r-1)/(b×r^k) + 1 ) )
```

Where: `b` = base price, `r` = growth rate, `k` = currently owned, `n` = buying, `c` = available currency.

### Multiplier Thresholds

Milestones at specific ownership counts (e.g. 25, 50, 100 owned) give large multiplier bonuses. This makes previously-optimal generators suddenly the best buy again — keeps player choices interesting.

---

## Part II — The Derivative Model

Generators produce *other generators* in cascading tiers instead of producing currency directly.

### Math

A chain of N tiers started from a single top-tier generator at time `t` grows as:

```
total = 1 + t + t²/2 + t³/6 + ... + t^N/N!
```

This approaches `e^t` as tiers increase — sub-exponential, but just barely. Allows exponential costs to still create a ceiling.

**Design implication:** Even at lower tiers, generators keep mattering because purchased counts boost the entire tier's production rate (not just the top).

---

## Part III — Prestige Math

Prestige currency earned on reset. Different games use different bases:

| Game | Formula | Basis |
|------|---------|-------|
| Realm Grinder | `p = (√(1 + 8·cM/10¹²) - 1) / 2` | Max currency ever |
| AdVenture Capitalist | `p = 150·√(cL/10¹⁵)` | Lifetime earnings |
| Cookie Clicker | `p = ∛(cL/10¹²)` | Lifetime earnings |
| Egg Inc | `Δp = (cR/10⁶)^0.14` | Current-run earnings |

**Lifetime-based:** Doubling your prestige gain requires 4x–8x more earnings (depending on square root vs cube root). Rewarding repeated runs.

**Run-independent (Egg Inc style):** Each run is isolated. Resetting at the same point gives the same reward — creates flat diminishing returns over time.

---

## Blackhole — Current Implementation

### Currency Source

Particles are absorbed into the blackhole. Reward per absorption:

```gdscript
# blackhole.gd:86
reward = absorb_reward × pow(radius / 8.0, absorb_exp)
# absorb_reward = 1.0,  absorb_exp = 2.0  (quadratic)
```

| Particle radius | Currency earned |
|----------------|----------------|
| 8 (base) | 1.0 |
| 11.3 (2 merged) | 2.0 |
| 16 (size 2) | 4.0 |
| 24 | 9.0 |
| 32 | 16.0 |

Bigger particles are **quadratically** more valuable. Merging before absorbing is the dominant strategy.

### Particle Merging

Area-conserving:
```gdscript
# particle.gd:55
new_radius = sqrt(r1² + r2²)
```

Two size-8 particles merge into radius ~11.3. This is the hidden "generator" — particles accumulate value through merging physics.

### Upgrade Cost Formula

```gdscript
# upgrade_menu.gd:79
cost = base_cost × pow(cost_mult, level)
```

This matches Part I's standard exponential cost model exactly.

| Upgrade | base_cost | cost_mult | Notes |
|---------|-----------|-----------|-------|
| spawn_rate | 10 | 2.0 | Gentlest growth |
| pulse_strength | 5 | 2.5 | |
| pulse_radius | 5 | 2.5 | |
| spawn_size | 25 | 2.8 | |
| pulse_frequency | 30 | 3.0 | Steepest |
| blackhole_pull | 10 | 3.0 | Steepest |
| event_horizon | 200 | 2.5 | Unlock gate |
| ship (unlock) | 500 | 2.5 | Unlock gate |

### Upgrade Effect Formula

Most upgrades use additive linear scaling:
```gdscript
# cursor.gd:31
strength = base × (1.0 + level × value_per_level)
```

This means each additional level adds the same *flat* percentage, but costs exponentially more. Effect gain decelerates against cost — creating the expected bottleneck.

### Prestige System (Planned — Not in Prototype)

Design intent: difficulty-forced prestige, not math-wall prestige.

- Enemies (mines, hazards) spawn progressively and escalate beyond the player's ability to manage
- This pressure — not a cost ceiling — is what triggers the reset decision
- On prestige: player picks **one permanent roguelike upgrade** (passive bonus, new mechanic, etc.)
- Each run is a slightly different build depending on what was chosen previously

This is a **stronger design than the article models**. The articles describe prestige as an escape from an exponential cost wall. Blackhole's version gives prestige *narrative and gameplay motivation* — you're not resetting because you're stuck, you're resetting to build a better run. Much closer to Egg Inc / Hades than Cookie Clicker.

---

## Gap Analysis

| # | Theory Concept | Blackhole Status | Notes |
|---|---------------|-----------------|-------|
| 1 | Exponential cost growth | ✅ Implemented | `base × mult^level` |
| 2 | Polynomial production growth | ✅ Implemented | Additive linear per level |
| 3 | Prestige / reset loop | 🔜 Planned | Difficulty-forced + roguelike pick on reset |
| 4 | Generator tiers (derivative model) | ❌ Missing | No cascading generators |
| 5 | Multiplier milestone bonuses | ❌ Missing | No 25/50-owned bonuses |
| 6 | Bulk purchase math | ❌ Missing | Buy 1 at a time only |
| 7 | Traceable CPS metric | ⚠️ Absent | Production is physics-based, not CPS |
| 8 | Quadratic particle rewards | ✅ Unique | Not in articles — Blackhole's own invention |
| 9 | Physics-merge accumulation | ✅ Unique | Particles as accumulating value objects |

---

## Recommendations

### 1. Prestige Loop (Planned — See Design Notes Above)

The difficulty escalation system (mines, hazards) will serve as the prestige trigger. When designed, consider:
- Hazard intensity curve should scale faster than upgrades can counter, ensuring prestige feels like a strategic retreat rather than a forced wall
- Roguelike pick pool should include things that change *how* you play (new mechanics) not just flat multipliers — that's what makes each run feel distinct
- Consider a soft prestige (keep some upgrades) vs hard prestige — the article math favors hard reset, but roguelike builds favor keeping some progression

### 2. Milestone Multipliers (Medium Priority)

Add threshold bonuses to the upgrade system. Example for `pulse_strength`:

```
Level 5  → ×2 bonus applied
Level 10 → ×3 bonus applied
Level 25 → ×5 bonus applied
```

This creates re-evaluation moments where previously-skipped upgrades suddenly look like the best buy.

### 3. Bulk Buy (Optional / Low Priority)

Formula from Part I exists if ever needed. Trivial to add. Not worth prioritising now.

### 4. `cost_mult` Values — Intentional Design

`blackhole_pull` at 3.0 is deliberate: it's the most powerful upgrade (passive gravity) and should stay expensive to prevent snowballing. `pulse_frequency` at 3.0 similarly gates a high-leverage mechanic. These are not bugs to fix.

### 5. Leverage the Unique Mechanics

The physics merge system and quadratic reward scaling are things none of the article models cover. These are Blackhole's competitive advantage. Design content around maximizing merge chains before absorption — this is the game's equivalent of optimizing a generator purchase order.
