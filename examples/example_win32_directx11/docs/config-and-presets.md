# glass.cfg — Configuration & Presets

How Glass Overlay loads, applies, captures, and persists its ~70 user-tunable
settings. The system was split out of the old single `config.inc` into
**`src/app/config/`** — `config.inc` is now just a 3-line aggregator that `#include`s:

- **`config/glass_config.inc`** — the `GlassConfig` struct + the global `g_cfg`.
- **`config/config_parse.inc`** — `CfgApply` (key→field, with clamps), `CfgParse`, and
  `CfgDefaultText` (the commented first-run template).
- **`config/config_io.inc`** — file read / path resolution / `LoadOrCreateConfig`, the
  preset path helpers, and `CfgSave` (the serializer).

The two bridge lambdas (`ApplyCfg` / `CaptureCfg`) now live in
**`src/app/config_apply.inc`** (still `#include`d textually *inside* `main()` so they can
capture the live UI locals by reference). The user-facing UI is in
**`src/app/pages/customize.inc`** (the Customize page: `CONFIG FILE`, `PRESETS`, `RESET`
panels).

---

## 1. Big picture

```
glass.cfg  ──CfgParse──▶  GlassConfig g_cfg  ──ApplyCfg──▶  live UI vars  ──▶  GlassCB (shaders.h)
   ▲                                              ▲   │
   └────────── CfgSave ◀── CaptureCfg ◀───────────┘   └──▶ rendered every frame
```

- **`glass.cfg`** is a plain `key = value` INI next to the executable. Comments start with `#`
  or `;`. Missing keys fall back to defaults; unparseable values are skipped; every numeric is
  **range-clamped** after parsing — a typo can never crash or break the app.
- **`GlassConfig g_cfg`** is the in-memory mirror of the file (one global instance).
- **Live UI vars** (`gBlur`, `eFresExp`, `accent[]`, …, all `main()` locals) are what the
  sliders edit and what feeds the shader constant buffer each frame. The config struct is *not*
  read every frame — it is a load/save container only.

### Safety contract (from the file header)

Every field default **equals the hard-coded look**, so a missing file behaves identically to
having no config. Writing is best-effort: failure is silently ignored. A fully-commented
template is written on first run.

---

## 2. `GlassConfig` struct (~70 keys)

Defined in `config/glass_config.inc`. Grouped, with the INI key, C++ member, default, and
clamp range. Clamp ranges come from `CfgApply` (in `config/config_parse.inc`, the per-key
validator).

### Window (geometry — *excluded* from presets, see §5)

| INI key | member | default | clamp |
|---|---|---|---|
| `window.width` | `win_w` | 1000 | 200..16384 |
| `window.height` | `win_h` | 760 | 200..16384 |
| `window.x` | `win_x` | -1 (center) | -1..16384 |
| `window.y` | `win_y` | -1 (center) | -1..16384 |
| `window.always_on_top` | `always_on_top` | 0 | 0..1 |
| `window.start_page` | `start_page` | 0 | 0..7 |

### Appearance

| INI key | member | default | clamp |
|---|---|---|---|
| `appearance` | `appearance` | 0 (dark) | 0..1 |
| `accent_r/g/b` | `accent_r/g/b` | 0.02 / 0.48 / 1.0 | 0..1 |
| `font_scale` | `font_scale` | 1.0 | 0.6..2.0 |
| `ui_scale` | `ui_scale` | 1.0 | 0.5..3.0 (see note) |
| `density` | `density` | 1 | 0..2 |

> **`ui_scale` is parsed and saved but pinned to 1.0 at apply time** — `ApplyCfg` hard-codes
> `uiScale = 1.0f` (scaling currently disabled). The key still round-trips through the file.

### Light (two independent rim lights)

| INI key | member | default | clamp |
|---|---|---|---|
| `light.angle` | `light_angle` | 341 | 0..360 |
| `light.intensity` | `light_intensity` | 1.0 | 0..3 |
| `light.angle2` | `light_angle2` | 161 | 0..360 |
| `light.intensity2` | `light_intensity2` | 0.0 (off) | 0..3 |

### Material (frosted body)

| INI key | member | default | clamp |
|---|---|---|---|
| `material.blur` | `mat_blur` | 0.55 | 0..1 |
| `material.opacity` | `mat_opacity` | 0.38 | 0..1 |
| `material.saturation` | `mat_saturation` | 1.35 | 0..4 |
| `material.refraction` | `mat_refraction` | 24.0 | 0..120 |
| `material.chroma` | `mat_chroma` | 2.5 | 0..30 |
| `material.edge` | `mat_edge` | 0.22 | 0..3 |
| `material.shadow` | `mat_shadow` | 0.10 | 0..1 |
| `material.flow` | `mat_flow` | 5.5 | 0..24 |

### Edge shader (`edge_*`) — the glass rim

| INI key | member | default | clamp |
|---|---|---|---|
| `edge.fresnel_exp` | `edge_fresnel_exp` | 2.5 | 0.5..8 |
| `edge.bevel_scale` | `edge_bevel_scale` | 0.6 | 0.1..1 |
| `edge.lip` | `edge_lip` | 0.16 | 0..1 |
| `edge.specular_exp` | `edge_specular_exp` | 40.0 | 4..200 |
| `edge.specular_amt` | `edge_specular_amt` | 1.05 | 0..4 |
| `edge.front_amt` | `edge_front_amt` | 1.0 | 0..4 |
| `edge.back_amt` | `edge_back_amt` | 0.85 | 0..4 |
| `edge.sat_pop` | `edge_sat_pop` | 1.4 | 0..4 |
| `edge.bevel_tilt` | `edge_bevel_tilt` | 2.6 | 0.5..6 |
| `edge.light_height` | `edge_light_height` | 0.8 | 0.2..2.5 |
| `edge.shadow` | `edge_shadow` | 0.45 | 0..2 |
| `edge.cursor_size` | `edge_cursor_size` | 170 | 40..400 |
| `edge.cursor_glow` | `edge_cursor_glow` | 0.0 | 0..3 |
| `edge.sheen` | `edge_sheen` | 1.0 | 0..3 |
| `edge.grain` | `edge_grain` | 1.0 | 0..4 |
| `edge.ambient_rim` | `edge_ambient_rim` | 0.55 | 0..2 |
| `edge.front_spread` | `edge_front_spread` | 100 (%) | 30..500 |
| `edge.back_spread` | `edge_back_spread` | 100 (%) | 30..500 |
| `edge.panel_br_spread` | `edge_panel_br_spread` | 100 (%) | 30..500 |
| `edge.panel_br_smooth` | `edge_panel_br_smooth` | 100 (%) | 30..400 |
| `edge.panel_rim_angle` | `edge_panel_rim_angle` | 135 (deg) | 0..360 |
| `edge.tbl_remove` | `edge_tbl_remove` | 0 | 0..1 |
| `edge.tbl_fill` | `edge_tbl_fill` | 0.6 | 0..2 |
| `edge.adapt_on` | `edge_adapt_on` | 0 | 0..1 |
| `edge.adapt_strength` | `edge_adapt_strength` | 0.35 | 0..1 |
| `edge.adapt_pivot` | `edge_adapt_pivot` | 0.60 | 0..1 |
| `edge.adapt_soft` | `edge_adapt_soft` | 0.30 | 0.02..0.6 |
| `edge.smooth_refraction` | `edge_smooth_refraction` | 0 | 0..1 |
| `edge.squircle_on` | `edge_squircle_on` | 0 | 0..1 |
| `edge.squircle_power` | `edge_squircle_power` | 5.0 | 2..8 |
| `edge.lens_amount` | `edge_lens_amount` | 0.0 | 0..1 |
| `edge.lens_power` | `edge_lens_power` | 2.0 | -4..4 |
| `edge.lens_a/b/c/d` | `edge_lens_a/b/c/d` | 0.7 / 2.3 / 5.2 / 6.9 | 0..5 / 0..6 / 0..6 / 0..10 |
| `edge.glow_weight` | `edge_glow_weight` | 0.0 | -1..1 |
| `edge.glow_phase` | `edge_glow_phase` | 0.5 | -6.3..6.3 |
| `edge.glow_edge0` | `edge_glow_edge0` | 0.5 | -1..1 |
| `edge.glow_edge1` | `edge_glow_edge1` | -0.5 | -1..1 |
| `edge.glow_bias` | `edge_glow_bias` | 0.0 | -1..1 |
| `edge.glow_anim` | `edge_glow_anim` | 0.0 | 0..4 |

### Effects (`fx_*`)

| INI key | member | default | clamp |
|---|---|---|---|
| `fx.sound` | `fx_sound` | 1 | 0..1 |
| `fx.parallax` | `fx_parallax` | 1 | 0..1 |
| `fx.cursor` | `fx_cursor` | 1 | 0..1 |
| `fx.auto_hide` | `fx_auto_hide` | 0 | 0..1 |
| `fx.show_fps` | `fx_show_fps` | 0 | 0..1 |

---

## 3. Load path — `LoadOrCreateConfig` + resolution order

`LoadOrCreateConfig` (in `config/config_io.inc`) is called once near the top of `main()`
(before the first `ApplyCfg`):

```cpp
LoadOrCreateConfig();   // main.cpp:195
```

It resolves the config path via **`CfgFindExisting()`**, in this order:

1. **`$GLASS_CONFIG`** — if the env var names a file that exists, use it verbatim.
2. **`<exe dir>\glass.cfg`** — the executable's own directory.
3. **Walk up to 6 parent directories** (`<exe>\..\glass.cfg`, `..\..`, … 6 levels) — handy
   when the exe is buried in `x64\Release\` but the config lives at the project root.

If a file is found it is read (≤1 MiB sanity cap in `CfgReadFile`) and parsed into `g_cfg`. If
**none** is found, a fully-commented default template (`CfgDefaultText()`) is written next to
the exe and `g_cfg` keeps its built-in defaults.

```cpp
static void LoadOrCreateConfig() {
    std::string path = CfgFindExisting();
    if (!path.empty()) { std::string t; if (CfgReadFile(path, t)) CfgParse(g_cfg, t); return; }
    std::string out = ExeDir() + "\\glass.cfg";   // none found -> write template
    FILE* f = fopen(out.c_str(), "wb");
    if (f) { std::string t = CfgDefaultText(); fwrite(t.data(),1,t.size(),f); fclose(f); }
}
```

### Parsing details (`CfgParse` → `CfgApply`)

`CfgParse` splits on newlines, strips `#`/`;` comments and whitespace, splits each line on the
first `=`, and calls `CfgApply(key, val)`. `CfgApply` is a long `if (key == "...")` chain: each
arm parses (`CfgParseF`/`CfgParseI`, which reject trailing junk) then clamps. **Unknown keys are
silently ignored**, so old/new configs are forward- and backward-compatible.

### The commented default writer — `CfgDefaultText()`

A big string literal that mirrors every key with its default and an inline comment. This is the
template a first-run user sees. **It is a hand-maintained duplicate of the struct defaults** —
see the hazard in §6.

> Note: `CfgDefaultText()` (first-run template) and `CfgSave()` (saved-from-UI output) are
> **two separate serializers**. The template is prose-rich; the save output is terse. Both must
> emit the same key set.

---

## 4. The bridge — `ApplyCfg` / `CaptureCfg` (`app/config_apply.inc`)

These two lambdas are defined in `src/app/config_apply.inc`, which is `#include`d textually
*inside* `main()` (so they can capture the live UI locals by reference) and are the *only*
place config values cross into/out of the running UI.

| | direction | used by |
|---|---|---|
| `ApplyCfg(const GlassConfig&)` | cfg → live UI vars | startup (`ApplyCfg(g_cfg)`), preset **Load**, slot **Load** |
| `CaptureCfg(GlassConfig) -> GlassConfig` | live UI vars → cfg | `Save Current Look`, preset slot **Save** |

```cpp
auto ApplyCfg = [&](const GlassConfig& c) {
    appearance = c.appearance; accent[0]=c.accent_r; ...
    gBlur = c.mat_blur; eFresExp = c.edge_fresnel_exp; ...
    uiScale = 1.0f;            // scaling disabled — pinned, ignores c.ui_scale
    Glass::SetSfxEnabled(fxSound);
};
auto CaptureCfg = [&](GlassConfig c) -> GlassConfig {  // takes by value, mutates, returns
    c.mat_blur=gBlur; c.edge_fresnel_exp=eFresExp; ...
    return c;                  // window geometry + start_page in `c` are passed through untouched
};
```

### The deliberate omission: window geometry + page

Both bridges **skip `win_w/h/x/y` and `start_page`** so loading a preset never moves the window
or jumps you off the Customize screen.

- `CaptureCfg` takes its `GlassConfig` **by value** and only overwrites the look fields — the
  window/page fields keep whatever the caller passed in (typically the live `g_cfg`).
- `ApplyCfg` never assigns `win_*` or `start_page` to live state. `start_page` is applied
  **once at startup only**: `page = g_cfg.start_page;` immediately after the first `ApplyCfg`.
- The only place geometry is captured is the `Save Current Look` button (§5), which writes the
  real window rect explicitly *after* calling `CaptureCfg`.

---

## 5. The Customize UI — `CONFIG FILE`, `PRESETS`, `RESET` (`app/pages/customize.inc`)

All three panels are in the `dispPage == 4` (Customize) branch, which is its own fragment:
`src/app/pages/customize.inc` (one of the `else if` arms `#include`d by `app/pages.inc`).

### `CONFIG FILE` → "Save Current Look"

Writes the main `glass.cfg`, **including** current window size/position:

```cpp
GlassConfig sc = CaptureCfg(g_cfg);              // all live look settings
sc.win_w=(int)((wr.right-wr.left)/scale); sc.win_h=(int)((wr.bottom-wr.top)/scale);
sc.win_x=wr.left; sc.win_y=wr.top;               // geometry only saved here
CfgSave(CfgSavePathStr(), sc); g_cfg = sc;
```

`CfgSavePathStr()` returns the config we loaded from, else `<exe>\glass.cfg`.

### `PRESETS` → 6 slots

Files are `glass_preset_1.cfg` … `glass_preset_6.cfg` next to the exe (`PresetPathStr(slot)`).
For each slot `1..6`:

```cpp
if (Glass::Button(sb, true))  CfgSave(PresetPathStr(ps), CaptureCfg(g_cfg));        // Save N
if (Glass::Button(lb, false) && exists) {                                           // Load N
    GlassConfig pc; if (LoadCfgFromPath(PresetPathStr(ps), pc)) ApplyCfg(pc);
}
// label shows "saved" / "empty" via FileExists(PresetPathStr(ps))
```

Because slots go through `CaptureCfg`/`ApplyCfg`, **presets carry the look but never the window
geometry or page** — loading a slot restyles the overlay in place. `LoadCfgFromPath` resets to
defaults first (`out = GlassConfig{}`) then parses, so a sparse preset file fills missing keys
cleanly.

### `RESET` → "Reset All Settings"

Resets the **live UI vars directly** (does *not* touch `g_cfg` or any file). It is the fourth
hand-maintained copy of the default set, and it includes extra live-only vars that aren't in
`GlassConfig` (e.g. `accentSel`, `animSpeed`, `reduceMotion`, `boldText`, `sidebarW`, `fxHaptics`,
`kbClicks`, `snapEdges`, …). **Watch:** a few values here intentionally differ from struct
defaults — e.g. `gRefr=22.0f` and `gEdge=0.0f` vs `mat_refraction=24.0`/`mat_edge=0.22` in the
struct. Reconcile deliberately, don't blindly copy.

---

## 6. ⚠️ HAZARD: the default key set is hand-duplicated in FOUR places

There is **no single source of truth** for the key list or its defaults. The same set is spelled
out independently in four locations, and they must stay byte-for-byte in agreement or settings
will silently fail to load, save, reset, or template correctly:

| # | Location | File | What breaks if you forget it |
|---|---|---|---|
| 1 | `GlassConfig` member + default | `config/glass_config.inc` (struct) | The value has no storage / wrong fallback |
| 2 | `CfgDefaultText()` literal | `config/config_parse.inc` | Key missing from first-run template; user never sees it |
| 3 | `CfgSave()` `snprintf` format **and** arg list | `config/config_io.inc` | Saved configs/presets drop the key (or, worse, format/arg drift = garbage output) |
| 4 | `RESET ALL` block | `pages/customize.inc` (`dispPage==4`) | "Reset All Settings" leaves the tunable at its last value |

`CfgApply()` (the parser arm, in `config/config_parse.inc`) is effectively a **fifth** spot:
without it, the key is read from disk but ignored.

### Checklist — when you add a tunable

1. **`config/glass_config.inc` — `GlassConfig`**: add the member + default.
2. **`config/config_parse.inc` — `CfgApply`**: add the `if (key == "...") { ...clamp... }` parse arm.
3. **`config/config_parse.inc` — `CfgDefaultText`**: add the commented line to the first-run template.
4. **`config/config_io.inc` — `CfgSave`**: add the format line **and** the matching `c.xxx` argument
   (keep format specifiers and the variadic arg order in lock-step).
5. **`pages/customize.inc` — RESET ALL** (`dispPage==4`): reset the live var to its default.
6. **`app/config_apply.inc` — `ApplyCfg` *and* `CaptureCfg`**: wire the live UI var both directions.
7. **`shaders.h` — `GlassCB` cbuffer** (only if the shader consumes it): add the field; the C++
   `GlassCB` struct in `glass.cpp` must match **byte-for-byte** (`static_assert(sizeof(GlassCB)
   == 416)`, plus a runtime HLSL-reflection size check that asserts on drift).
8. **`pages/customize.inc` — the page slider/toggle**: add the actual `Glass::Slider(...)`/checkbox
   that edits the live var on the Customize page.

Miss any one and you get a partial feature with no compile error — the value just won't persist,
reset, or render.
