# Glass Overlay

A real-time **glassmorphic** desktop overlay for Windows. The entire window is a
borderless, per-pixel-alpha sheet of frosted glass — it captures your live desktop
(DXGI Desktop Duplication), blurs it with a dual-Kawase chain, and renders a fully
glassmorphic, spring-animated UI on top via a custom HLSL material. Built from
scratch in C++ on **Direct3D 11 + Dear ImGui**.

> **Demo:** _add a screenshot or GIF here_ — e.g. `![Glass Overlay](docs/demo.gif)`

## Features

- **Live blurred backdrop** — real desktop behind the UI, captured and blurred every frame (dual-Kawase).
- **Custom HLSL glass material** — SDF shapes, refraction, Fresnel edge-lighting, chromatic aberration, squircle corners, twin-lobe bevel; ~50 tunable parameters.
- **Spring-driven motion** — a damped mass-spring system drives every animation: morphing metaball selection, gel-on-press, spring color glides.
- **118 demo "surfaces"** plus a full glass widget kit (toggles, sliders, segmented controls, modals, command palette, a live color picker…).
- **Tunable performance** — runs up to your monitor's refresh (240 Hz) with **independent UI-fps and blur-fps** controls.
- **Built-in recorder** — one-key MP4 capture (1440p, up to 240 fps, H.264) that also records the app's own UI sound effects.

## Build

```sh
bash scripts/build.sh        # build Release | x64
bash scripts/build.sh run    # build, then launch
```

Requires **Visual Studio 2022** (MSBuild, toolset v143) and the **Windows 10 SDK**
(10.0.22621.0). The project lives inside the Dear ImGui example tree because it
references the ImGui sources, backends, FreeType, and `data/` relative to the
repository root.

## Controls

| Key | Action |
|-----|--------|
| `Esc` | Close the overlay |
| `F9` | Start / stop screen recording |
| `F11` | Toggle screen-capture visibility (OBS / Snipping Tool); freezes the backdrop |
| `Cmd`/`Ctrl` + `K` | Command palette |

## Layout

```
.
├── example_win32_directx11.vcxproj   MSBuild project (Release | x64, toolset v143)
├── README.md  ·  LICENSE  ·  DISCLAIMER.md  ·  THIRD-PARTY-NOTICES.md
├── docs/                             architecture & subsystem notes
├── scripts/build.sh                  build (and optionally run)
└── src/
    ├── main.cpp                      entry point, window + frame loop, app shell
    ├── app/                          application layer (text-included into main.cpp)
    │   ├── pages/                    settings pages
    │   ├── config/                   glass.cfg load / apply / save
    │   ├── render/                   D3D device, SSAA resolve, MP4 recorder
    │   └── megasettings.inc          data-driven settings catalog
    └── glass/                        the renderer
        ├── glass.{h,cpp}             material params, Renderer, widget API
        ├── backdrop.{h,cpp}          desktop capture + dual-Kawase blur chain
        ├── shaders.h                 blur + glass HLSL
        ├── spring.h                  spring-damper animation
        ├── surfaces.{h,cpp}          app-surface registry + launchpad / dock
        ├── widgets/                  widget implementations (#included into glass.cpp)
        └── surfaces/                 demo app-surfaces (#included into surfaces.cpp)
```

The `.inc` files are textual includes, not separately compiled units: each is
`#include`d into a single `.cpp`, preserving file-static linkage while keeping the
sources browsable. Only the `.cpp`/`.h` files appear in the project.

## License

MIT — see [LICENSE](LICENSE). Third-party components retain their own licenses
([THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md)).

## Credits & disclaimer

Created by **poncipp** (GitHub [@pondot](https://github.com/Pondot)).
This is an independent glassmorphic UI rendering demo. See [DISCLAIMER.md](DISCLAIMER.md)
for trademark and non-affiliation notes.
