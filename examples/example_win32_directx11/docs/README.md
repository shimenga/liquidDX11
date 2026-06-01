# Glass Overlay documentation

A transparent, always-on-top **glassmorphic** settings overlay for Windows
(Win32 + Direct3D 11 + Dear ImGui). Real-time frosted-glass material with
glassmorphic edge lighting, springy spring-based widget motion, 118 demo "surfaces",
a live config system, and a built-in MP4 recorder.

## Start here

| Doc | What it covers |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | The big picture: the unity-build model, folder map, and the per-frame data flow (config → state → shader → present). **Read this first.** |
| [BUILD.md](BUILD.md) | How to build (MSBuild, x64 Release) and run; runtime deps (fonts, ffmpeg) and env overrides. |

## Subsystems

| Doc | What it covers |
|---|---|
| [glass-material.md](glass-material.md) | The renderer + HLSL shaders: the `GlassCB` cbuffer and the full `edge_p0..p11` parameter map, materials, the dual-Kawase backdrop blur, and the glass edge model. |
| [animation.md](animation.md) | The spring system (`Spring`/`SpringCache`/`SpringStyle`), `SmoothCol` color glide, and the buttery spring-based motion recipe. |
| [widgets.md](widgets.md) | The immediate-mode widget library in `src/glass/widgets/` and its strict include order. |
| [surfaces.md](surfaces.md) | The 118 demo surfaces, the `g_surfaces[]` registry, and the index-coupling fragility. |
| [config-and-presets.md](config-and-presets.md) | `glass.cfg` persistence, the `ApplyCfg`/`CaptureCfg` bridge, presets — and the four-place "default key set" sync hazard. |
| [recording-and-capture.md](recording-and-capture.md) | The D3D device lifecycle, the F9 MP4 recorder, and the SSAA resolve pass. |

## Source layout (one line each)

```
src/
  main.cpp                  app entry: window, D3D bootstrap, the render loop
  app/
    ui_state.inc            main()-local live UI state (sliders/toggles)   [#included in main]
    config_apply.inc        ApplyCfg / CaptureCfg (cfg <-> live state)      [#included in main]
    config.inc → config/    glass.cfg: struct / parse+defaults / io+save
    pages.inc  → pages/      dashboard page bodies (one file per page)      [#included in main]
    megasettings.inc        the data-driven settings catalog
    render_capture.inc → render/   device / MP4 recorder / SSAA resolve
  glass/
    glass.{h,cpp}           Glass::Renderer, materials, GlassCB
    shaders.h               kBlurHLSL (backdrop blur) + kGlassHLSL (glass PS)
    spring.h                Spring / SpringCache / SpringStyle
    backdrop.{h,cpp}        DXGI desktop-duplication capture + dual-Kawase blur
    widgets/                the immediate-mode widget library (.inc, ordered)
    surfaces.{h,cpp} + surfaces/   the 118 demo mini-apps + registry
```

> **Unity build:** only the four `.cpp` (`main`, `glass`, `surfaces`, `backdrop`) are
> compiled; every `.inc` is `#include`d in a **dependency-significant order** and shares
> file-static linkage within its translation unit. See [ARCHITECTURE.md](ARCHITECTURE.md).
