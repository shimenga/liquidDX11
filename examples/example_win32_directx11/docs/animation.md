# The Motion System

Every "buttery" movement in the overlay — a button squashing under a press, a toggle knob smearing into a gooey trail, an icon brightening on hover — is driven by one small primitive: a damped mass-spring. This doc explains the spring, how springs are cached per widget, how colors glide, the spring-motion recipe that composes these into a tactile feel, and the one frame-delta cap that keeps it all from going slow-motion.

Source of truth:
- `src/glass/spring.h` — `Spring`, `SpringStyle`, `SpringCache`
- `src/glass/widgets/widgets_core.inc` — `SmoothCol`, and the canonical widgets (`Button`, `Toggle`, `Slider`, `RowToggle`, `RowValue`) that use springs
- `src/glass/glass.cpp` — `Dt()`, `US()`, the live `SpringCache` behind `g->springs()`
- `src/glass/widgets/widgets_misc.inc` — `SubmitBlob` (metaball trails)
- `src/main.cpp` — the frame-delta cap (the `CAP long frames` line, ~`main.cpp:370`)

---

## 1. The `Spring` struct

A `Spring` is a 1-D damped harmonic oscillator: a unit `mass` on a spring of `stiffness`, with viscous `damping`. State is three floats; everything else is a tuning constant.

```cpp
struct Spring {
    float x = 0.0f;        // current value (what you render)
    float v = 0.0f;        // current velocity
    float target = 0.0f;   // where it's pulling toward
    float stiffness = 280.0f;
    float damping   = 0.0f; // 0 => auto-derive critical damping on first Tick
    float mass      = 1.0f;
};
```

You drive a spring by setting `target` each frame and calling `Tick(dt)`. You read `x` to position/scale/colorize whatever you're drawing. `v` (velocity) is also useful directly — knobs read `|v|` to stretch in the direction of travel.

### Sub-stepped integration

`Tick` is a semi-implicit (symplectic) Euler integrator. The force is the classic `F = -k·(x - target) - c·v`; velocity is updated first, then position uses the new velocity (this is what makes it stable and not explicit-Euler-divergent):

```cpp
void Tick(float dt) {
    if (damping <= 0.0f) damping = 2.0f * std::sqrt(stiffness * mass); // lazy critical default
    const int steps = (dt > 1.0f/120.0f) ? 2 : 1;  // sub-step at low fps
    const float h = dt / (float)steps;
    for (int i = 0; i < steps; ++i) {
        const float F = -stiffness * (x - target) - damping * v;
        v += (F / mass) * h;
        x += v * h;
    }
}
```

Two details matter:

- **Lazy critical damping.** If `damping` is still 0 on the first tick, it's set to `2·√(k·m)`, the critical value. Most springs are created via `Spring::Make` (below), which sets damping explicitly, so this only fires for raw `Spring{}` instances.
- **Sub-stepping.** A stiff spring integrated with one big step at low fps can overshoot or ring. When `dt > 1/120 s` (i.e. below 120 fps, which is almost always), `Tick` runs **two** half-steps. This halves the effective `h`, keeping stiff springs (CriticalFast at `k=450`) accurate without a fixed-timestep accumulator.

`Snap(to)` teleports: sets `x = target = to` and `v = 0`. Use it to initialize a spring to a known state with no animation (e.g. a toggle that should start already-on).

---

## 2. `SpringStyle` — the four presets

`Spring::Make(style, start)` builds a spring whose `x` and `target` both start at `start`, with stiffness/damping chosen per style. Damping is expressed as a multiple of the critical damping ratio `2·√(k·m)`:

| Style | Stiffness | Damping (× critical) | Behavior | Use for |
|---|---|---|---|---|
| `CriticalFast` | 450 | 1.0 (critical) | Snaps to target fast, **no overshoot** | Quick state changes that must feel instant |
| `Critical` | 280 | 1.0 (critical) | Settles cleanly, **no overshoot** | The default — state changes, hover lift, row scale, toggle position |
| `Bouncy` | 320 | 0.55 (under-damped) | Overshoots and oscillates back | Playful taps — button press/hover scale |
| `Overdamped` | 220 | 1.6 (over-damped) | Lags, eases in slowly, never overshoots | Gooey trailing tails — metaball lag positions |

Rules of thumb baked into the codebase:

- **`Critical` for anything stateful.** A switch flipping, a row lifting, a selection pill moving — you never want it to overshoot past the real value. `RowToggle`/`RowValue` use `Critical` for both the row scale and the switch position, with explicit comments: *"press squash, hover lift; NO overshoot."*
- **`Bouncy` for taps.** A button's scale spring (`Button`, slot 1) and a slider knob's grow spring use `Bouncy` so the press squash springs back with a soft jiggle. The overshoot is the point — it reads as a soft gel.
- **`Overdamped` for trailing tails.** The gooey "smear" behind a moving knob is a second position that **lags** the real one. `Overdamped` (damping 1.6×) makes the tail ease in slowly and never catch up abruptly, so the metaball blob (see §5) stays stretched while sliding and reforms when it settles.

```cpp
static Spring Make(SpringStyle s, float start) {
    Spring sp; sp.x = start; sp.target = start;
    switch (s) {
        case CriticalFast: sp.stiffness=450; sp.damping=2*sqrt(k*m);        break;
        case Critical:     sp.stiffness=280; sp.damping=2*sqrt(k*m);        break;
        case Bouncy:       sp.stiffness=320; sp.damping=0.55*2*sqrt(k*m);   break;
        case Overdamped:   sp.stiffness=220; sp.damping=1.6 *2*sqrt(k*m);   break;
    }
    return sp;
}
```

---

## 3. `SpringCache` — persistent state keyed by (id, slot)

Widgets are immediate-mode: the `Button` function runs from scratch every frame, so it cannot own its spring as a local. The spring's `x`/`v` must survive across frames. `SpringCache` is that storage.

```cpp
struct SpringCache {
    std::unordered_map<uint64_t, Spring> map;
    Spring& Get(uint32_t imgui_id, uint32_t slot, SpringStyle style, float initial = 0.0f) {
        const uint64_t key = ((uint64_t)imgui_id << 32) | (uint64_t)slot;
        auto it = map.find(key);
        if (it == map.end()) {
            map.emplace(key, Spring::Make(style, initial)); // first touch: construct with style+initial
            return map.find(key)->second;
        }
        return it->second; // subsequent frames: same spring, with last frame's x/v
    }
}
```

The key packs two 32-bit values into a `uint64_t`:

- **`imgui_id`** — the ImGui ID of the widget (`ImGui::GetID(label)`), making the spring unique per-widget. Two buttons with different labels get independent springs.
- **`slot`** — a small integer distinguishing *which* spring within that one widget. A button needs several (scale, elevation, ripple, label color), so each gets its own slot.

The live cache is reachable from any widget via `g->springs()`. `style` and `initial` are **only honored on first touch** (when the entry is constructed); on every later frame the cached spring is returned as-is and you re-set `target` yourself.

### Slot-numbering convention

Slots are a per-widget namespace. The convention:

- **Low slots (1–9): a widget's core motion.** These were the original springs. E.g. in `Toggle`: slot 2 = knob position, slot 3 = grab-grow, slot 4 = the Overdamped lagging tail. In `Slider`: slot 4 = knob grow, slot 5 = the Overdamped tail.
- **High slots (20+): added by the buttery motion pass.** When the "buttery" lift/squash/color-glide layer was added on top of existing widgets, it claimed slots 20+ to avoid colliding with the original low-slot springs. In `RowToggle`/`RowValue`: slot 20 = row scale, slot 21 = the hover plate elevation, slot 22 = chevron nudge (or knob grab), slot 23 = knob lag, and **24/28/32/36 = `SmoothCol` color springs** (each consumes a 4-slot block — see §4).

So when adding a new animated detail to an existing widget, pick an unused high slot and leave a comment. Reusing a slot already in use by that widget silently shares the spring.

---

## 4. `SmoothCol` — per-channel color glide

Colors snap ugly. `SmoothCol` eases an `ImU32` toward a target color by running **four springs**, one per RGBA channel, so a hover-brighten or selection-tint glides like lit glass instead of flipping.

```cpp
ImU32 SmoothCol(uint32_t id, uint32_t slot, ImU32 target, SpringStyle st = SpringStyle::Critical) {
    // unpack target into R,G,B,A (0..255 floats)
    Spring& sr = g->springs().Get(id, slot+0, st, tr); sr.target = tr; sr.Tick(Dt());
    Spring& sg = g->springs().Get(id, slot+1, st, tg); sg.target = tg; sg.Tick(Dt());
    Spring& sb = g->springs().Get(id, slot+2, st, tb); sb.target = tb; sb.Tick(Dt());
    Spring& sa = g->springs().Get(id, slot+3, st, ta); sa.target = ta; sa.Tick(Dt());
    return IM_COL32(clamp(sr.x), clamp(sg.x), clamp(sb.x), clamp(sa.x)); // re-clamp to 0..255
}
```

Key facts an engineer must internalize:

- **It consumes 4 consecutive slots** (`slot+0..slot+3`). That's why the row widgets space their `SmoothCol` calls 4 apart: 24, 28, 32, 36. Never start a `SmoothCol` block fewer than 4 slots from another spring on the same widget.
- **Defaults to `Critical`** so colors settle without overshooting into out-of-gamut values; the per-channel `x` is re-clamped to 0–255 anyway.
- **Forward-declared at the top of `widgets_core.inc`** because `Button` (defined first) calls it for its secondary-label color glide, while the definition lives ~190 lines below:

```cpp
// top of widgets_core.inc:
ImU32 SmoothCol(uint32_t id, uint32_t slot, ImU32 target, SpringStyle st = SpringStyle::Critical);
```

Typical use: `RowValue` glides the icon, label, value, and chevron colors with four separate `SmoothCol(id, 24/28/32/36, ...)` calls so each brightens independently on hover.

---

## 5. The buttery spring-motion recipe

These are the composition patterns. Each is just springs + a render primitive (`g->Submit` a `Primitive`, or `SubmitBlob` for a metaball). All sizes pass through `US(v)` (UI-scale) so the motion scales with the global UI scale.

### Hover-lift / press-squash

Set a scale spring's target by state, then scale the geometry by `spring.x`. Use `Critical` for rows (no overshoot) or `Bouncy` for tappable buttons (a little jiggle).

```cpp
// RowToggle / RowValue — Critical, no overshoot:
Spring& rscl = g->springs().Get(id, 20, SpringStyle::Critical, 1.0f);
rscl.target = act ? 0.98f : (hov ? 1.015f : 1.0f);  // press squash, hover lift
rscl.Tick(Dt());
// contents scale about the row center, so icon+label+switch lift together:
float rs = rscl.x;
auto sx = [&](float x){ return rowCx + (x - rowCx) * rs; };
```

A companion `rele` spring (slot 21) raises a faint frosted "plate" behind the row on hover and sinks it on press (`elevate`). Crucially, only the *visual* geometry is scaled — the `InvisibleButton` hit-rect and the ImGui cursor advance are unchanged, so layout/clicks stay stable while the pixels breathe.

### Selection-pop

A moving selection (toggle knob, segmented pill) uses two springs: a `Critical` **position** spring (slot 2 in `Toggle`) that tracks the on/off amount `kc ∈ [0,1]`, and a `Bouncy`/`Critical` **grow** spring (grab) that scales the knob up while pressed:

```cpp
Spring& k = g->springs().Get(id, 2, SpringStyle::Critical, *v ? 1.0f : 0.0f);
k.target = *v ? 1.0f : 0.0f; k.Tick(Dt());      // slide position
Spring& grab = g->springs().Get(id, 3, SpringStyle::Critical, 1.0f);
grab.target = active ? 1.12f : 1.0f; grab.Tick(Dt()); // grow on grab
```

The knob also reads **velocity** to stretch in the direction of travel — `speed = min(0.45, |k.v|·0.06)` widens it along the slide and shortens it perpendicular, a liquid "velocity stretch."

### Pixel-accurate drag

The `Slider` rule (commented in source): **while dragging, the knob tracks the mouse 1:1 — never a spring.** A spring under the finger would feel laggy. The shown position `tShown` is set directly to the mouse-derived `t` while `active`; the spring-style glide is only applied **on release**, and even then via an exponential ease, not the value the cursor is on:

```cpp
if (active) tShown = t;                                  // 1:1, no lag in-hand
else { tShown += (t - tShown) * Clampf(Dt()*18.0f, 0,1); // glide only when settling
       if (fabsf(tShown - t) < 0.0005f) tShown = t; }
```

### Metaball blob trails (`SubmitBlob`)

The gooey "smear" behind a moving knob/pill is a **second, lagging position** rendered as a metaball: two rounded-rects merged with a smooth-min in the shader. The head is the real (pixel-accurate) position; the tail is an `Overdamped` spring that lags behind. When they're far enough apart, submit a blob that connects them; when they've reconverged, submit a single rounded-rect (cheaper, sharper).

```cpp
Spring& lag = g->springs().Get(id, 4, SpringStyle::Overdamped, kc); // tail lags the head
lag.target = kc; lag.Tick(Dt());
float kxh = /* head: pixel-accurate */, kxt = /* tail: from lag.x */;
if (std::fabs(kxh - kxt) > 2.0f)
    SubmitBlob(kxh, ky, khw, khh,  kxt, ky, khw*0.82f, khh*0.82f,  radius, /*k=*/9.0f, 1.0f, Material::Knob);
else { /* settled: submit a plain rounded-rect knob */ }
```

`SubmitBlob` (in `widgets_misc.inc`) sizes a single draw quad to cover both shapes (padded by `k`), stashes both rect centers/half-sizes plus the smooth-min strength `blob_k` into the `Primitive`, and submits it; the fragment shader does the metaball merge. The same pattern drives segmented-control pills and the side-nav highlight (`widgets_extra.inc`, `widgets_misc.inc`, `surfaces/part6.inc`).

---

## 6. `Dt()` and the frame-delta cap (critical)

Springs are time-stepped, so they depend entirely on `dt`. The widgets read it through a one-liner:

```cpp
static inline float Dt() { return ImGui::GetIO().DeltaTime; } // glass.cpp
```

Two failure modes that the **frame-delta cap in `main.cpp`** (the `CAP long frames` line,
~`main.cpp:370`) prevents:

```cpp
if (io.DeltaTime > 0.05f) io.DeltaTime = 1.0f / 30.0f;  // CAP long frames
```

- **The first-frame explosion.** Frame 0's `DeltaTime` spans all of D3D11/shader/backdrop initialization — possibly *seconds*. Fed into a `Bouncy` entrance or any widget spring, that single huge step makes the spring overshoot wildly and ring for seconds (the historic "flickers a few times on open" bug). Capping it kills the blow-up.
- **Low-fps slow-motion.** Note the cap clamps to `1/30 s`, **not** to `1/60`. If a frame genuinely took 80 ms, resetting `dt` to `1/60` would make springs advance less than real time elapsed — they'd play in slow motion and drift out of sync with the wall clock. Clamping to `1/30` keeps springs tracking real time during a hitch while still bounding any single step. Combined with `Spring::Tick`'s internal sub-stepping (§1), a capped-but-still-large `dt` stays stable.

The cap is applied once per frame, right after `ImGui::NewFrame()`, so **every** `Dt()` call downstream (and `SmoothCol`, and any direct `io.DeltaTime` easing in `main.cpp`/pages) sees the clamped value.
