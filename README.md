# Glass Overlay

Glass Overlay is a Windows desktop UI demo built with C++17, Direct3D 11, and Dear ImGui 1.92.8. It renders a borderless per-pixel-alpha overlay, captures the live desktop with DXGI Desktop Duplication, blurs it with a dual-Kawase chain, and draws a glassmorphic Dear ImGui interface on top.

This repository keeps the upstream Dear ImGui source layout so the example can reference ImGui, the Win32/DX11 backends, FreeType, and local assets with simple relative paths. The main project lives here:

```text
examples/example_win32_directx11
```

## Features

- Real-time blurred desktop backdrop.
- Custom HLSL glass material with refraction, edge lighting, chromatic aberration, rounded SDF shapes, and tunable shader parameters.
- Spring-driven UI motion for toggles, buttons, panels, and demo surfaces.
- A gallery of demo surfaces plus reusable glass widgets.
- Optional MP4 capture through `ffmpeg`.
- Local configuration and presets through `glass.cfg`.

## Build

Requirements:

- Windows
- Visual Studio 2022 with the C++ workload
- MSBuild toolset v143
- Windows 10 SDK 10.0.22621.0 or newer

From `examples/example_win32_directx11`:

```bat
MSBuild example_win32_directx11.vcxproj /p:Configuration=Release /p:Platform=x64
Release\example_win32_directx11.exe
```

Or from a Git Bash shell:

```sh
bash scripts/build.sh
bash scripts/build.sh run
```

The project is x64-only because the bundled FreeType import library is x64.

## Repository Layout

```text
backends/                         Dear ImGui Win32/DX11 backends
data/                             Fonts and generated font data used by the demo
examples/example_win32_directx11/ Glass Overlay app, docs, scripts, and project file
misc/freetype/                    Dear ImGui FreeType integration
thirdparty/freetype/              FreeType headers and x64 import library
imgui*.cpp, imgui*.h              Dear ImGui core sources
```

## Docs

- [Example README](examples/example_win32_directx11/README.md)
- [Build details](examples/example_win32_directx11/docs/BUILD.md)
- [Architecture](examples/example_win32_directx11/docs/ARCHITECTURE.md)
- [Third-party notices](examples/example_win32_directx11/THIRD-PARTY-NOTICES.md)
- [Disclaimer](examples/example_win32_directx11/DISCLAIMER.md)

## License

Original project code in `examples/example_win32_directx11/src`, docs, scripts, and project files is MIT licensed. Dear ImGui, FreeType, and bundled fonts/icons retain their own licenses; see the linked third-party notices.
