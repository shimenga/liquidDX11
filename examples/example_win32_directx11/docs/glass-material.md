# Glass Material — Renderer + Shader

This document covers the Liquid-Glass material renderer (`src/glass/glass.cpp` +
`glass.h`) and its two HLSL programs (`src/glass/shaders.h`). It is the authoritative
reference for the constant-buffer contract, the `Material` table, and the glass pixel
shader's edge model. Read this to reach full understanding of how a glass primitive
gets from a `Submit()` call to lit pixels.

---

## 1. Pipeline at a glance

The glass effect is a two-stage GPU pass:

1. **Backdrop blur** (`kBlurHLSL`): the captured desktop is reduced with a
   dual-Kawase down/up-sample chain into **two** SRVs — a *heavy* (frosted) blur and a
   *soft* (lighter) blur.
2. **Glass compositing** (`kGlassHLSL`): every glass primitive draws a single quad
   (6 verts, no vertex/index buffers — `Draw(6,0)` with `nullptr` input layout). The
   pixel shader samples the two blurs, lenses/refracts them, tints, and lights the edge.

`Renderer::Render(heavy, soft)` binds both blur SRVs to `t0`/`t1`, the shared constant
buffer to `b0` (VS + PS), and draws every queued `Primitive` then every overlay one.
Output is **premultiplied alpha** — the PS returns `glass * a` in rgb, so the blend state
is `SrcBlend=ONE, DestBlend=INV_SRC_ALPHA`.

```
queue_ (Submit) ┐
                ├─► draw_one(): Map cb_ → fill GlassCB → Draw(6,0)
overlay_ (Submit│Overlay) ┘
```

---

## 2. The `GlassCB` constant buffer — byte-match + 416-byte guard

The PS is driven entirely by one cbuffer at `register(b0)`. It is declared **twice**:
once in HLSL (`shaders.h`, inside `kGlassHLSL`) and once as a C++ POD (`glass.cpp`,
`struct GlassCB`). These two declarations **must be field-for-field, byte-for-byte
identical** — `draw_one()` does a raw `memcpy`-style fill of the mapped buffer, so any
drift silently corrupts every uniform downstream of the misaligned field.

Two guards enforce this:

| Guard | Location | What it checks |
|---|---|---|
| Compile-time | `static_assert(sizeof(GlassCB) == 416, ...)` | C++ struct is exactly 416 bytes |
| Run-time (reflection) | `CompileShaders()` after `D3DCompile` | HLSL cbuffer size == `sizeof(GlassCB)` |

The runtime guard reflects the compiled PS and compares the *HLSL* cbuffer size to the
C++ struct size, catching a mismatch that the `static_assert` alone cannot (it only sees
the C++ side):

```cpp
ID3D11ShaderReflection* refl = nullptr;
if (SUCCEEDED(D3DReflect(pb->GetBufferPointer(), pb->GetBufferSize(),
                         __uuidof(ID3D11ShaderReflection), (void**)&refl)) && refl) {
    if (auto* rcb = refl->GetConstantBufferByName("GlassCB")) {
        D3D11_SHADER_BUFFER_DESC sbd = {};
        if (SUCCEEDED(rcb->GetDesc(&sbd)) && sbd.Size != (UINT)sizeof(GlassCB))
            IM_ASSERT(false && "GlassCB layout drift: shaders.h cbuffer != glass.cpp struct");
    }
}
```

The buffer is `D3D11_USAGE_DYNAMIC` and re-mapped (`MAP_WRITE_DISCARD`) per primitive, so
every glass quad can carry different geometry/material values from one shared buffer.

### 2.1 Fixed (non-edge) cbuffer fields

The first part of `GlassCB` carries per-draw geometry and the active material. HLSL field
→ C++ field, in declaration order (HLSL `float2` packs as `float[2]`; HLSL packing rules
keep each scalar group on its 16-byte boundary because the field layout was authored to do
so):

| HLSL | C++ | Meaning |
|---|---|---|
| `window_size` | `window_size[2]` | backbuffer/client px |
| `desktop_size` | `desktop_size[2]` | full desktop px (UV reference) |
| `window_origin` | `window_origin[2]` | window top-left on desktop |
| `light_dir` | `light_dir[2]` | normalized light 1 (y down) |
| `widget_center` | `widget_center[2]` | client px |
| `widget_half_size` | `widget_half_size[2]` | px |
| `corner_radius, fade, blur_mix, saturation` | same | radius / master alpha / blur blend / vibrancy |
| `tint_color` | `tint_color[4]` | rgb tint (a unused) |
| `tint_opacity, brightness, contrast, grain` | same | milky tint + tone |
| `refr_strength, refr_band, highlight, inner_shadow` | same | refraction + lit/far rim |
| `border_intensity, border_width, shadow_strength, shadow_radius` | same | hairline rim + drop shadow |
| `shadow_offset` | `shadow_offset[2]` | px (positive y = down) |
| `time, quad_pad` | same | clock + VS quad expansion (shadow room) |
| `tilt` | `tilt[2]` | environment tilt (motion + cursor) |
| `sheen, chroma` | same | gloss + rim dispersion px |
| `cursor` | `cursor[2]` | pointer client px (specular spotlight) |
| `_pad_cur` | `_pad_cur[2]` | **reused**: `.x` = liquid-flow amount (px); `.y` unused |
| `blob1_center/half, blob2_center/half` | `[2]` each | metaball A/B (smooth-min merge) |
| `blob_k` + `_pad_blob` | `blob_k`, `_pad_blob[3]` | smin radius (0 = disabled) |

`quad_pad` is computed in `draw_one()` from shadow radius/offset + border width so the
VS expands the quad enough to contain the soft drop shadow.

---

## 3. The edge-param block — `edge_p0..edge_p11`

After the fixed fields, the cbuffer carries **12 float4 registers** of live,
slider-tunable Liquid-Glass edge parameters (Customize page / `glass.cfg`). They are
populated from `GlassEdgeConfig edge_cfg_` plus the two light directions/intensities in
`draw_one()`. The C++ struct names them `edge0..edge11`; the HLSL names them
`edge_p0..edge_p11`.

This is the **exact** packing (each row = one `float4`; `(spare)` = unused, written 0):

| Reg | .x | .y | .z | .w |
|---|---|---|---|---|
| `edge_p0` | `fresnel_exp` | `bevel_scale` | `lip` | `specular_exp` |
| `edge_p1` | `specular_amt` | `front_amt` | `back_amt` | `sat_pop` |
| `edge_p2` | `bevel_tilt` | `light_height` | `edge_shadow` | `cursor_size` |
| `edge_p3` | `cursor_glow` | `sheen_amt` | `grain_amt` | `ambient_rim` |
| `edge_p4` | `front_spread` | `back_spread` | `panel_br_spread` | `panel_br_smooth` |
| `edge_p5` | `light_dir2.x` | `light_dir2.y` | `light1_amt` | `light2_amt` |
| `edge_p6` | `panel_rim_dir.x` | `panel_rim_dir.y` | `tbl_fill` | `(spare)` |
| `edge_p7` | `adapt_strength` | `adapt_pivot` | `adapt_soft` | `smooth_refraction` |
| `edge_p8` | `squircle_power` | `lens_amount` | `lens_power` | `lens_a` |
| `edge_p9` | `lens_b` | `lens_c` | `lens_d` | `glow_weight` |
| `edge_p10` | `glow_phase` | `glow_edge0` | `glow_edge1` | `glow_bias` |
| `edge_p11` | `glow_anim_speed` | `(spare)` | `(spare)` | `(spare)` |

Notes on fields that are **derived**, not copied verbatim from `GlassEdgeConfig`:

- `edge_p5.xy` (`light_dir2`) comes from `light2_angle_deg_` via `sin/-cos`, not the cfg.
- `edge_p5.zw` are `light1_amt_` / `light2_amt_` (from `SetLights`), not the cfg.
- `edge_p6.xy` (`panel_rim_dir`) is `panel_rim_angle` (deg) converted to a unit vector.

Because the edge params are **constant-buffer values** (uploaded per draw), the shader is
compiled **once** with no `#define`/macros — sliders take effect immediately with no
recompile. `GlassEdgeConfig`'s defaults equal the stock look.

---

## 4. The Material system

A glass primitive selects one `Material` enum; the renderer looks up its `MaterialParams`
and copies them into the fixed cbuffer fields in `draw_one()`.

```cpp
enum class Material : uint8_t { Thin, Regular, Thick, Accent, Knob, Clear };
```

`MaterialParams` (glass.h) bundles the per-material constants — frost/vibrancy/tint/
refraction/edge/shadow/sheen/chroma. The table `kMaterials[]` (glass.cpp) holds one row
per enum value:

| Material | Role | Notable params |
|---|---|---|
| `Thin` | `.thinMaterial` — light frost, lots of backdrop detail | low `blur_mix` 0.28, high `saturation` 1.65, low tintA 0.30 |
| `Regular` | default cards | `blur_mix` 0, strong `highlight` 1.0, large `refr` 40 |
| `Thick` | modal panels, heavy frost | `blur_mix` 0.58, tintA 0.52 |
| `Accent` | vivid system-tint fill (toggle-on, slider fill, primary btn) | blue tint, tintA 0.90, high `sheen` 0.80 |
| `Knob` | bright control lens (toggle knob, slider thumb) | near-white tint, tintA 0.94, `highlight` 0.72 |
| `Clear` | neutral pass-through (full-window live desktop for recording) | everything zeroed |

Accessors / mutators:

- `Params(m)` / `EditParams(m)` — read / live-edit a material row (config page).
- `SetGlobalMaterial(blur, opacity, sat, refr, chroma, edge, shadow)` — applies to
  `Thin/Regular/Thick` at once (the global appearance sliders).
- `SetAccent(r,g,b)` — recolors the Accent row's tint.
- `SetAppearance(0|1)` — dark/light: flips ink colors and retints
  `Thin/Regular/Thick/Knob` (saving/restoring originals in `g_matDefaults`). It only
  touches `tint_rgb` + `brightness`, so per-frame `SetGlobalMaterial` still composes.

In `draw_one()` the shadow params are scaled by `p.elevate` (hover lift), and
`shadow_strength` is clamped to 0.9.

---

## 5. The HLSL shaders (`shaders.h`)

Both programs are embedded as raw-string literals and runtime-compiled with `D3DCompile`
(no fxc / no .cso files). `kGlassHLSL` is compiled twice — once as `vs_5_0` (entry
`VS_Glass`) and once as `ps_5_0` (entry `PS_Glass`).

### 5.1 `kBlurHLSL` — dual-Kawase blur

A full-screen-triangle VS (`VSFull`) plus two pixel passes:

- **`PSDown`** — 5-tap box down-sample (center×4 + 4 diagonal half-texel taps, ÷8).
- **`PSUp`** — 8-tap tent up-sample (÷12).

Run iteratively at shrinking/growing resolution this produces a smooth, gaussian-like
frost far cheaper than a mip chain (Marius Bjorge, SIGGRAPH 2015). `texel` = `1/texSize`.

### 5.2 `kGlassHLSL` — split across two raw-string literals

> **Critical build detail.** MSVC limits a single string literal to ~16 KB. `kGlassHLSL`
> exceeds that, so it is written as **two adjacent raw-string literals that the compiler
> concatenates** at the seam:
>
> ```cpp
> inline constexpr const char* kGlassHLSL = R"HLSL(
> ... cbuffer, VS_Glass, sdRoundedRect, hash21, sampleBackdrop*, sdScene, DirLight ...
> )HLSL" R"HLSL(
> float4 PS_Glass(VSOut i) : SV_Target { ... }
> )HLSL";
> ```
>
> The split point is the literal text `)HLSL" R"HLSL(` between `DirLight` and `PS_Glass`.
> It is a *string* boundary only — there is **one** logical HLSL source. When editing,
> keep the seam between complete top-level declarations; never split mid-function.

The VS (`VS_Glass`) builds a 6-vert quad from `widget_center ± (widget_half_size +
quad_pad)`, converts to NDC, and passes `local` (px from center) + `client` (client px) to
the PS.

---

## 6. The glass edge model (`PS_Glass`)

The PS is the heart of the look. Pixel flow, in order:

### 6.1 SDF rounded-rect with optional squircle corners

`sdRoundedRect(p, hs, r)` returns the signed distance to a rounded rectangle. The corner
norm is selectable: `n = max(edge_p8.x, 2.0)` (squircle power). `n == 2` is the classic
**circular** (L2) corner; `n > 2` is a continuous-curvature **squircle** via an
Lⁿ-norm `pow(pow(qx,n)+pow(qy,n), 1/n)`.

```hlsl
float n = max(edge_p8.x, 2.0);
float corner = (n <= 2.001) ? length(qm)
                            : pow(pow(qm.x, n) + pow(qm.y, n), 1.0 / n);
return min(max(q.x, q.y), 0.0) + corner - r;
```

`sdScene()` wraps it and, when `blob_k > 0`, smooth-min-blends two rounded-rects into a
metaball (glass merge). All downstream geometry (shape mask, normal, bevel,
shadow) uses `sdScene`.

### 6.2 Soft contact shadow + early discard

A second SDF evaluation offset by `shadow_offset` builds a diffuse drop shadow
(`saf*saf*shadow_strength`). Pixels outside both shape and shadow are `discard`ed.

### 6.3 3D bevel normal + Fresnel rim

The 2D SDF gradient (central differences, `e=1.5`) gives the outward normal `nrm`. A rim
height profile (`rimT² = slope`, concentrated within a size-adaptive `bevelW`) tilts a
true **3D bevel normal** `N3 = normalize(float3(nrm*slope*bevel_tilt, 1))` — flat top is
`+z`, the rim domes outward. The **continuous Fresnel rim** is
`fresR = pow(saturate(1 - N3.z), fresnel_exp)` — a bright lip the whole way around, not a
drawn stroke.

### 6.4 Refraction, chromatic dispersion, frost, and the magnify lens

- **Edge refraction**: `refr_px = nrm * (edge²) * refr_strength * refScale` displaces the
  backdrop UV at the rim. `refScale = clamp(44/sizeMin, 0.35, 1.7)` bends small controls
  hard, big panels gently.
- **Smooth refraction** (`edge_p7.w`): eases the falloff with a smootherstep and adds
  extra rim-band blur for a creamy edge; `0` is a no-op.
- **Whole-surface magnify lens** (`edge_p8.y` amount, `edge_p8.z` power, `lens_a..d` in
  `edge_p8.w`/`edge_p9.xyz`): an exponential `f(dist)` profile
  `1 - b*(c*e)^(-d*x-a)` scales the *entire* backdrop radially about the panel center —
  content bulges through the glass. Negative power = concave/invert. `0` = off.
- **Liquid flow** (`_pad_cur.x`): a slow multi-frequency domain warp so the frost breathes.
- **Chromatic dispersion** (`chroma`): R/G/B sampled along `±cdir` so the rim splits color.
- **Frosted taps**: near the rim, samples are biased toward the heavy blur
  (`sampleBackdropFrost`) and 4 diagonal taps are blended in proportionally.

### 6.5 Vibrancy, adaptive opacity, tint, grain

Saturation is applied (with **extra rim-confined `sat_pop`** gated by `fresR` so the lip
"pops"), then contrast/brightness. **Adaptive opacity** (`edge_p7.x/.y/.z`) raises the
milky tint over bright backdrops (per-pixel `smoothstep` on bg luminance) so the glass
stays legible; `0` = off. Then the milky `tint_color` is mixed at `opEff`, a faint
luminosity gradient toward the light is added, and near-static film grain (`grain_amt`).

### 6.6 Twin-lobe directional rim lighting (2 lights)

`DirLight(L, nrm, N3, np, fresR, band)` computes one light's contribution and is called
**twice** — light 1 (`light_dir + tilt`) scaled by `light1_amt` (`edge_p5.z`) and light 2
(`edge_p5.xy + tilt`) scaled by `light2_amt` (`edge_p5.w`). Each light produces:

- A **twin-lobe rim**: `fr = max(0,l)^e` (near/reflection lobe, `front_amt`/`front_spread`)
  plus `bk = max(0,-l)^e` (far/refraction-exit lobe, `back_amt`/`back_spread`). The
  `max(0,·)^e` form decays smoothly **to zero** at the cross-diagonal, so the lit rim
  fades progressively to the unlit corner with no hard cut.
- A tight **specular hotspot** (`specular_exp`, `specular_amt`) from the 3D half-vector,
  with light elevation `light_height`.
- A directional **gloss/sheen** (`sheen * sheen_amt`).

The all-around lip is `fresR * (lip + border_intensity*ambient_rim)`, gated on big
elements toward the lights (`dcov`) so cross-diagonal corners don't carry leftover white.

### 6.7 Panel BR reach, far-rim dip, TR/BL corner fill

- **Panel/button BR rim reach** (`panel_br_spread`/`panel_br_smooth`, location
  `panel_rim_dir` in `edge_p6.xy`): on panels (gated by `pbf`) a *positional* gradient
  brightest at the chosen corner extends the rim along the edges (a panel's straight edge
  has a constant normal, so only a positional gradient reads as "reach"). Confined to the
  rim `band`.
- **Far-rim depth dip**: `fresR*(1-f1)*(1-b1)*inner_shadow*edge_shadow` darkens the dim
  cross-diagonal rim, smoothly (matches the lobes).
- **TR/BL corner fill** (`edge_p6.z = tbl_fill`): `(1-f1)(1-b1)` is 1 at the two
  cross-diagonal corners and 0 at the lit ones, so adding light there lifts them out of
  shadow. `0` = off.

### 6.8 Cursor spotlight + adaptive opacity + angular glow

- **Pointer spotlight** (`cursor_size`, `cursor_glow`): a soft gaussian highlight
  following the cursor, stronger at the rim. `cursor_glow = 0` = off.
- **Angular edge-glow** (`glow_weight` in `edge_p9.w`; phase/edges/bias in `edge_p10`;
  anim speed `edge_p11.x`): a directional sheen lobe keyed to polar angle
  `atan2(np.y,np.x)`, confined to the rim band via `smoothstep(edge0,edge1,distN)`, with
  optional time-driven phase drift so a glint sweeps around the panel. `0` = off.

### 6.9 Composite

```hlsl
float a    = shape * fade;                 // glass coverage
float outA = a + shadow_a * (1.0 - a);     // glass over shadow
return float4(glass * a, outA);            // premultiplied
```

`adapt`/opacity is `saturate(tint_opacity + adapt)`; final `glass` is `saturate`d before
the premultiplied output.

---

## 7. Editing checklist (don't break the contract)

When you add or move an edge param:

1. Add the field to `GlassEdgeConfig` (glass.h) with a stock-look default.
2. Pack it into the right `edge_pN.[xyzw]` slot in `draw_one()` (glass.cpp).
3. Read it from the matching slot in `kGlassHLSL` (shaders.h) — keep both comments in sync.
4. If you changed the cbuffer *size*, update the `static_assert(... == 416)` and verify
   the runtime `D3DReflect` guard still passes (it will assert otherwise on first compile).
5. If editing near the literal seam, keep `)HLSL" R"HLSL(` between whole declarations.
