# Glass Overlay Architecture

## What this app is

A **transparent, always-on-top glass settings overlay** for Windows. It is a single-window Win32 + Direct3D 11 program built on **Dear ImGui 1.92.8**. The whole window is a frameless, per-pixel-alpha sheet of glass: there is no opaque background. Behind every panel the user sees their **live desktop**, captured via DXGI Desktop Duplication and blurred in real time, so the UI reads like a frosted-glass material sitting on top of whatever is on screen.

The visible UI is a glassmorphic dashboard: a login / "hello" intro, a sidebar-navigated settings app (General / Display / Battery / Storage / **Customize**), an app grid, a bottom dock, plus **118 demo "surfaces"** (Weather, Music, Calendar, Calculator, …) that exist to show off the glass material. The **Customize** page exposes the renderer's edge/light/material knobs as live sliders; the **megasettings** catalog adds thousands more data-driven toggles. None of this persists app state beyond `glass.cfg` — it is a showcase + tuning harness for the glass renderer.

---

## Unity build model

This project does **not** compile every `.cpp`. Only **four** project-owned translation units are compiled (plus the stock ImGui core / backends / freetype, per `example_win32_directx11.vcxproj`):

| Compiled TU | Role |
|---|---|
| `src/main.cpp` | Win32 window, D3D11 swapchain, the per-frame loop, all UI state, config glue |
| `src/glass/glass.cpp` | The glass `Renderer` (shaders, cbuffer upload, draw) + the widget library |
| `src/glass/surfaces.cpp` | The 118 demo screens + Launchpad + Dock |
| `src/glass/backdrop.cpp` | DXGI desktop duplication + dual-Kawase blur |

Everything else is a **`.inc` fragment** that is `#include`d *textually* into one of those four TUs. Fragments are **not** independently compiled and have **no header guards** — they are raw source pasted at a specific point.

### Why unity build

* **File-static linkage / one shared namespace.** The widget library and surfaces lean heavily on `static` helpers (`Clampf`, `DrawIcon`, `TextC`, `SmoothCol`, `PlaySfx`, …) and a single `Glass::` namespace. Pasting fragments into one TU lets every widget see those statics directly — no headers, no `extern`, no link-time surface. Each of the four TUs re-declares its own file-local copies of the small helpers it needs (e.g. `surfaces.cpp` redefines `Clampf`/`Dt` because `glass.cpp`'s statics are invisible across TUs).
* **Include ORDER is load-bearing.** Fragments are pasted in **dependency order**; a fragment may use anything defined above it. Reordering the `#include`s breaks the build.

### The four include graphs

**`main.cpp`** — the application TU:
```
main.cpp
 ├─ app/megasettings.inc      (near top: DrawMegaSettings() + the settings catalog)
 ├─ app/config.inc            (3-line aggregator: GlassConfig, g_cfg, LoadOrCreateConfig/CfgSave — glass.cfg I/O)
 │     ├─ config/glass_config.inc (the GlassConfig struct + global g_cfg)
 │     ├─ config/config_parse.inc (CfgApply key->field, CfgParse, CfgDefaultText template)
 │     └─ config/config_io.inc    (file read / path resolution / LoadOrCreateConfig / presets / CfgSave)
 ├─ app/ui_state.inc          (#included INSIDE main(), ~main.cpp:327 — every main()-local UI state var)
 ├─ app/config_apply.inc      (#included INSIDE main(), ~main.cpp:328 — ApplyCfg/CaptureCfg bridge lambdas)
 ├─ app/pages.inc             (#included INSIDE main()'s loop — draws the sidebar dashboard pages)
 └─ app/render_capture.inc    (#included AFTER main(): D3D device/RTV + recorder + SSAA)
       ├─ render/device_d3d11.inc   (device/swapchain/RTV lifecycle; pulls in <d3dcompiler.h>)
       ├─ render/recorder_mp4.inc   (F9 in-app ffmpeg MP4 recorder)
       └─ render/ssaa_resolve.inc   (supersampling resolve pass; uses d3dcompiler from above)
```
`device_d3d11.inc` **must precede** `ssaa_resolve.inc` (it includes the `<d3dcompiler.h>` that the resolve pass needs). `pages.inc` is unusual: it is `#include`d **inside the render loop body** (around `main.cpp:746`), so it is just the body of the dashboard, with full access to the loop's locals (`page`, `scale`, all the `static` UI state).

**`glass.cpp`** — the renderer + widget library TU. After the `Renderer` and material/cbuffer code, it pastes the widget library in strict order (`glass.cpp:663`):
```
glass.cpp
 ├─ shaders.h                 (kGlassHLSL string: backdrop blur + the glass PS/VS)
 └─ widgets/  (ORDER MATTERS)
     ├─ audio.inc             (PlaySfx — must be first; everyone calls it)
     ├─ widgets_core.inc      (Button, Toggle, Slider, DrawIcon, SmoothCol, rows — the primitives)
     ├─ widgets_input.inc     (Segmented, Checkbox, Radio, Stepper, Chip, Dropdown, panels)
     ├─ fields_color.inc      (ColorButton, color picker, text/search/password fields)
     ├─ widgets_misc.inc      (SidebarNav, ProgressRing, Badge, Avatar, Rating, Spinner)
     ├─ modals.inc            (Alert, ActionSheet, ContextMenu, CommandPalette)
     ├─ widgets_extra.inc     (TabBar, DynamicIsland, Scrubber, ActivityRings, Gauge, Calendar)
     └─ chrome.inc            (Banner, ShareSheet, ControlCenter, Onboarding, MenuBar, SwipeRow)
```

**`surfaces.cpp`** — the demo screens TU. Defines shared drawing helpers (`RoundedGrad`, `ImgOrGrad`, `SurfaceHit`, `RowWash`), then pastes the six surface parts, then the `g_surfaces[]` dispatch table:
```
surfaces.cpp
 └─ surfaces/part1.inc … part6.inc   (the 118 Draw_* functions; table at surfaces.cpp:147)
```

**`backdrop.cpp`** — standalone; no fragments.

> Rule of thumb: a `.cpp` is a real TU; a `.inc` is a paste. To find where a symbol lives, look at the `#include` order of the owning TU, not at a header.

---

## Folder map (`src/`)

```
src/
  main.cpp                 Win32 + D3D11 host, frame loop, ALL live UI state, config glue
  glass/                   the glass renderer + shaders + widget library (TU: glass.cpp)
    glass.cpp/.h           Renderer class, MaterialParams table, GlassCB cbuffer, BeginCard/EndCard
    shaders.h              kGlassHLSL: dual-Kawase blur shaders + the glass VS/PS
    backdrop.cpp/.h        DXGI Desktop Duplication -> dual-Kawase blur -> heavy/soft SRVs
    spring.h               critically-damped / bouncy spring helper (widget animation)
    surfaces.cpp/.h        118 demo screens + app grid + bottom dock (TU: surfaces.cpp)
    widgets/               the widget library, pasted into glass.cpp (audio→core→…→chrome)
    surfaces/              part1..part6.inc — the 118 Draw_* demo-screen bodies
  app/                     application-layer fragments pasted into main.cpp
    config.inc             3-line aggregator -> config/ (glass.cfg load/save + commented default)
    config/                glass_config.inc (struct+g_cfg), config_parse.inc (CfgApply/CfgParse/CfgDefaultText), config_io.inc (LoadOrCreateConfig/presets/CfgSave)
    ui_state.inc           every main()-local slider/toggle/page state var (pasted inside main(), ~327)
    config_apply.inc       ApplyCfg/CaptureCfg cfg<->live-state bridge lambdas (pasted inside main(), ~328)
    pages.inc              aggregator -> pages/ (the sidebar dashboard pages, pasted inside main()'s loop)
    pages/                 general/display/battery/storage/customize/profile/account/launchpad/surface.inc
    megasettings.inc       data-driven global-settings catalog + DrawMegaSettings()
    render_capture.inc     aggregator: device + recorder + SSAA
    render/                device_d3d11.inc, recorder_mp4.inc, ssaa_resolve.inc
```

---

## Per-frame data flow

The renderer is **immediate-mode**: ImGui widgets and surface code don't paint glass directly — they **`Submit()` primitives** (rounded rects tagged with a `Material`) into the renderer's queue during the frame, and the renderer draws them all at the end against the blurred backdrop. Config → live state → renderer → cbuffer → shader → present:

```
glass.cfg ──LoadOrCreateConfig()──▶ g_cfg ──ApplyCfg()──▶ live UI vars in main()  (sliders/toggles)
                                                              │  (Customize page edits these directly)
                                                              ▼  every frame:
                       SetAppearance / SetAccent / SetGlobalMaterial / SetDensity / SetLiquidFlow
                       g_glass_renderer.SetEdgeConfig(GlassEdgeConfig) / SetLights(...)
                                                              │
   widgets + surfaces ──Submit(Primitive{material,rect,…})──▶ Renderer queue_ / overlay_
                                                              │
   Backdrop.Capture() ─DXGI dup → dual-Kawase─▶ heavySRV(), softSRV()
                                                              │
                       Renderer.Render(heavySRV, softSRV)  ── per primitive: Map cbuffer (GlassCB)
                                                              │   from MaterialParams + edge_cfg_ + lights,
                                                              │   bind 2 backdrop SRVs, Draw(6) a quad
                                                              ▼
                       glass PS samples BlurHeavy/BlurSoft at the refracted UV → frosted pixel
                                                              │
                       ImGui_ImplDX11_RenderDrawData (text/icons) drawn on top
                                                              ▼
                       g_pSwapChain->Present(1,0)  → DWM composites our per-pixel-alpha sheet
```

### 1. Config → live state
`LoadOrCreateConfig()` reads `glass.cfg` next to the exe (writing a commented default on first run), filling the global `GlassConfig g_cfg`. `main()` calls **`ApplyCfg(g_cfg)`** once at startup, which pushes every field into the loop's `static` UI variables (`gBlur`, `eFresExp`, `accent[3]`, `appearance`, …). `ApplyCfg` deliberately does **not** touch window size/pos or the current page, so loading a preset never moves the window. `CaptureCfg()` is the inverse, used by the Presets / save path. The **Customize** page mutates those same live vars directly via `Glass::Slider`.

### 2. Live state → renderer (each frame, `main.cpp` ~600–645)
The loop folds accessibility toggles into the material, then calls the renderer's setters:
```cpp
Glass::SetAppearance(appearance);                 // Light/Dark: flips ink + retints frost
Glass::SetAccent(accSm[0], accSm[1], accSm[2]);   // eased accent color (Accent material tint)
Glass::SetGlobalMaterial(eBlur,eOp,eSat,eRefr,eChroma,eEdge,eShadow);  // Thin/Regular/Thick
Glass::SetLiquidFlow(reduceMotion ? 0 : gFlow);   // animated backdrop warp
g_glass_renderer.SetEdgeConfig(ec);               // ~40 Fresnel/bevel/lens/glow edge knobs
g_glass_renderer.SetLights(eLightAngle, eLight1Amt, eLight2Angle, eLight2Amt);  // 2 rim lights
```
`SetGlobalMaterial`/`SetAccent` mutate the global **`kMaterials[]`** table (the `Thin/Regular/Thick/Accent/Knob/Clear` tuning rows). `SetEdgeConfig`/`SetLights` store into the renderer's `edge_cfg_` / light angles — applied at draw time.

### 3. Renderer → cbuffer → shader (`Renderer::Render`, `glass.cpp:490`)
For each submitted `Primitive`, `Render` maps the dynamic **`GlassCB`** constant buffer and fills it from that primitive's material (`Params(p.material)`), the live `edge_cfg_`, and the two light directions (`edge5[]`). `GlassCB` is **416 bytes** and must match the HLSL `cbuffer GlassCB` in `shaders.h` byte-for-byte; this is enforced two ways — a C++ `static_assert(sizeof(GlassCB)==416)` and a **runtime shader-reflection check** in `CompileShaders()` that asserts the reflected cbuffer size equals `sizeof(GlassCB)`. The two backdrop SRVs are bound to `t0`/`t1`, and each primitive is one `Draw(6, 0)` (a quad clipped to the primitive's scissor rect). The PS samples `BlurHeavy`/`BlurSoft` at a refracted UV and applies tint / saturation / Fresnel-bevel edge + twin-lobe rim light to produce the frosted pixel.

### 4. Backdrop (`backdrop.cpp` / `backdrop.h`)
`Backdrop::Capture()` runs every frame (kept live so the desktop keeps moving, even under recording). It acquires the latest desktop frame via **DXGI Desktop Duplication** (GPU-side, no CPU readback), copies it full-res, then runs a **dual-Kawase** down/up blur chain (`down_[6]` at /2…/64, `up_[5]`) producing two pre-blurred full-res textures:
* **`heavySRV()`** — wide frosted blur (the material's tint band)
* **`softSRV()`** — moderate blur (the detail band)

These are passed straight into `Renderer::Render(...)` and sampled by the glass PS. Because the overlay window is `WDA_EXCLUDEFROMCAPTURE`, the duplicated frame shows the desktop **behind** our window rather than feeding our own output back into itself.

### 5. Present
`ImGui::Render()` builds draw data; the glass primitives are rendered first (optionally into an offscreen **SSAA** target then box-resolved to the backbuffer), then `ImGui_ImplDX11_RenderDrawData` paints text/icons on top, then `g_pSwapChain->Present(1,0)`. The backbuffer is cleared to **fully transparent** (`{0,0,0,0}`), so DWM composites only the pixels the glass actually wrote.

---

## Win32 / window plumbing (the "it's actually transparent" bits)

These live at the top of `main()` and are what make the overlay a true glass sheet:

* **Single-instance mutex.** `CreateMutexW(L"Local\\GlassOverlaySingleInstance")` — a second copy would spawn another always-on-top layered window that fights the first for z-order and leaks into its desktop-duplication backdrop (reads as flicker). On `ERROR_ALREADY_EXISTS` it foregrounds the existing window and exits.
* **`WS_EX_LAYERED` + DWM per-pixel alpha.** The window is created `WS_POPUP | WS_EX_LAYERED`. `SetLayeredWindowAttributes(..., LWA_ALPHA, 255)` plus **`DwmExtendFrameIntoClientArea(hwnd, {-1})`** (a "sheet of glass" margin of -1) makes DWM honor the backbuffer's per-pixel alpha — transparent pixels show the desktop through.
* **`WDA_EXCLUDEFROMCAPTURE` (`0x11`).** `SetWindowDisplayAffinity(hwnd, 0x11)` keeps the overlay out of its own desktop duplication (and out of OBS / screenshots). **F11** toggles a "screenshot mode" that drops this to `WDA_NONE` so capture tools can record the overlay; **F9** forces it back on while the built-in MP4 recorder runs.
* **Frame-delta clamp.** The first frame's `dt` spans all of D3D/shader/backdrop init (seconds); the loop caps `io.DeltaTime` at `1/30` so the entrance spring and widget springs don't explode into multi-second oscillation.

### Other global loop behaviors
* `g_cfg.start_page` sets the initial page once at startup (presets don't change page).
* Keyboard: `Ctrl+,` → Customize, `Ctrl+1..5` → pages, `Ctrl+0` → Launchpad, `Ctrl+L` → Light/Dark, `Ctrl/Cmd+K` → command palette; right-click → glass context menu.
* SSAA (Tier-3 supersampling) is wired but defaults **off** (`ssaaOn=false`) — it softened the glass and dropped fps.

---

## Mental model in one paragraph

`main.cpp` owns the window and the frame loop and holds every piece of live UI state as function-`static`s. Each frame it (a) refreshes the blurred desktop backdrop, (b) pushes the live tuning state into the global material table + the renderer's edge/light config, (c) runs ImGui — where widgets (`glass.cpp`'s library) and demo surfaces (`surfaces.cpp`) **`Submit()`** glass primitives instead of painting them, then (d) the `Renderer` draws every queued primitive against the two blurred backdrop textures by filling the 416-byte `GlassCB` from each primitive's material, and (e) ImGui text/icons are layered on top and the transparent backbuffer is `Present`ed for DWM to composite. The whole thing is one unity build: four `.cpp` TUs, with the bulk of the code pasted in via order-sensitive `.inc` fragments so the widget/surface code shares file-static helpers within a single `Glass::` namespace.
